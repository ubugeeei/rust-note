# Arena allocation

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_arena`, `compiler/rustc_middle/src/arena.rs`
- **Date:** 2026-06-15

## Question

What does `compiler/rustc_arena` provide, how do `TypedArena` and `DroplessArena` differ, and how does the `declare_arena!` machinery wire dozens of types' arenas into `TyCtxt` so the compiler can hand out `&'tcx`/`&'hir` references instead of boxing every node?

## Short answer

An arena is a bump allocator: allocation is a pointer bump, individual objects are never freed, and the whole arena (with everything in it) is dropped at once (`compiler/rustc_arena/src/lib.rs:3`). rustc has two flavors: `TypedArena<T>` for a single type that needs `Drop`, and `DroplessArena` for any mix of `Copy`/non-drop types, which skips all destructors (`lib.rs:108`, `:348`). The `declare_arena!` macro builds one `Arena<'tcx>` struct holding one `DroplessArena` plus one `TypedArena` per listed droppable type, routing each `alloc` to the right sub-arena (`lib.rs:612`). `TyCtxt`/`GlobalCtxt` own these arenas (`arena` and `hir_arena`) for their whole lifetime, which is exactly why interned types and HIR nodes can be passed around as cheap `&'tcx`/`&'hir` references (`compiler/rustc_middle/src/ty/context.rs:707`).

## Why a compiler wants a bump allocator

A compiler creates an enormous number of small, immutable, graph-shaped objects (types, HIR nodes, MIR bodies) that all share one lifetime — they live until the crate is compiled, then die together. Three wins:

- **Cheap allocation** — "bump a pointer and write" (`lib.rs:142` for `TypedArena::alloc`, `:424` for `DroplessArena::alloc_raw`), vs a general allocator call per object.
- **Free all at once** — no per-object `free`/`Drop` bookkeeping; the arena's `Drop` reclaims every chunk (`lib.rs:312`). The module doc: arenas "destroy the objects within, all at once" and don't support deallocating individual objects (`lib.rs:3`).
- **Stable `&'tcx` references** — because the arena (owned by `GlobalCtxt`) outlives every object in it, an allocation is returned as a borrow tied to the arena's lifetime. That's what lets rustc thread `Ty<'tcx>`, `&'tcx [...]`, `&'hir ...` everywhere instead of `Box`/`Rc`.

### Before / after intuition

```rust
// Box-per-node: N allocations, N frees, ownership baked into the tree.
enum Expr { Lit(i64), Add(Box<Expr>, Box<Expr>) }

// Arena: bump-allocate, hand out &'a, drop everything together at the end.
enum Expr<'a> { Lit(i64), Add(&'a Expr<'a>, &'a Expr<'a>) }
let arena = DroplessArena::default();
let one = arena.alloc(Expr::Lit(1));
let e   = arena.alloc(Expr::Add(one, arena.alloc(Expr::Lit(2)))); // &'a Expr<'a>
```

This mirrors rustc's real shape: HIR and `ty` nodes carry an `'tcx`/`'hir` lifetime and are reached through references into a `TyCtxt`-owned arena.

## `TypedArena<T>` vs `DroplessArena`

**`TypedArena<T>`** (`lib.rs:108`) holds one type. Keeps `ptr`/`end` cursors and a `Vec<ArenaChunk<T>>`. `alloc` bumps `ptr` and `ptr::write`s the object (`:142`). It *does* run destructors: `Drop` walks every chunk and `assume_init_drop`s live elements (`:312`), guarded by `mem::needs_drop::<T>()` so non-drop types skip linear teardown (`:68`). Use it when `T` owns resources.

**`DroplessArena`** (`lib.rs:348`) holds many types at once, as long as each is `Copy`/non-drop. `alloc`/`alloc_slice`/`alloc_str` all `assert!(!mem::needs_drop::<T>())` (`:461`,`:485`,`:504`), and there is **no `Drop` for contents** — dying frees chunks only. Two details: it **bump-allocates downward** from `end` (slightly faster, `:356`), and stays aligned to `DROPLESS_ALIGNMENT = align_of::<usize>()` (`:346`) so LLVM can elide alignment fixups.

## How chunks grow

Both share `ArenaChunk` (`lib.rs:39`), a leaked `Box<[MaybeUninit<T>]>` whose `Drop` frees the box. Growth is geometric with a cap: first chunk `PAGE = 4096` bytes, each next **double** until `HUGE_PAGE = 2 MiB`, then constant (`:100`). See `TypedArena::grow` (`:256`) / `DroplessArena::grow` (`:386`). The comment notes it "scales well, from arenas that are barely used up to arenas used for 100s of MiBs", matching Linux page/huge-page sizes.

## The `declare_arena!` machinery and `TyCtxt`

`declare_arena!` is a `decl_macro` (`lib.rs:612`). Given a list of `name: ty` entries it generates:

- A struct `Arena<'tcx>` with one `dropless: DroplessArena` plus one `TypedArena<$ty>` per listed type (`:614`).
- An `ArenaAllocatable` trait selected by marker types `IsCopy`/`IsNotCopy` (`:620`): every `T: Copy` allocates in `dropless` (`:631`); each listed non-`Copy` type checks `needs_drop` *at allocation time* and routes to `dropless` if it doesn't need drop, else to its dedicated `TypedArena` (`:647`). (A non-drop listed type leaves its `TypedArena` empty.)
- `alloc`, `alloc_slice`, `alloc_str`, `alloc_from_iter` methods (`:672`).

The *list* of types lives in `arena_types!` macros. `rustc_middle`'s (`compiler/rustc_middle/src/arena.rs:6`) names the big payloads — `mir::Body`, `TypeckResults`, `Steal<Thir>`, `OwnerNodes`, layout/`FnAbi`/`AdtDefData` — and invokes the macro at `:130`. `[decode]` tags generate decoder impls for the incremental on-disk cache.

`GlobalCtxt` **owns** the arena for the whole compilation: `pub arena: &'tcx WorkerLocal<Arena<'tcx>>` and `pub hir_arena: &'tcx WorkerLocal<hir::Arena<'tcx>>` (`compiler/rustc_middle/src/ty/context.rs:707`). `WorkerLocal` gives each parallel worker its own arena. Because `tcx` lives as long as `'tcx`, anything `tcx.arena.alloc(...)`s comes back as `&'tcx`.

## HIR arena and the interning relationship

HIR has its **own** arena, declared by a separate `arena_types!` in `rustc_hir` (`compiler/rustc_hir/src/arena.rs`). AST→HIR lowering allocates into `tcx.hir_arena` (`compiler/rustc_ast_lowering/src/lib.rs:186`), so lowered nodes become `&'hir`.

Interning sits *on top of* the arena. `CtxtInterners` (`context.rs:134`) pairs each interned kind with a hash set plus the shared `arena`. `intern_ty` looks the `TyKind` up; on a miss it bump-allocates into the arena and stores the resulting `&'tcx` as the canonical copy (`context.rs:209`). So the arena provides stable backing storage and the interner guarantees one shared `&'tcx` per distinct value — together that makes `Ty<'tcx>` a cheap, `Copy`, pointer-comparable handle.

## See also

- [[10-performance-and-robustness]] · [[19-hir-format]] · [[20-mir-format]] · [[06-reading-the-compiler]] · [[07-glossary]]
