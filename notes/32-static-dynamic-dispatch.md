# Static vs dynamic dispatch

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_middle` (vtable), `compiler/rustc_codegen_ssa`, `compiler/rustc_trait_selection`
- **Date:** 2026-06-15

## Question

When you call a trait method, how does the compiler decide *which* function runs — and what is the difference, in rustc's data structures, between generics/`impl Trait` (static) and `dyn Trait` (dynamic)?

## Short answer

Static dispatch (`fn f<T: Trait>(x: T)`, `impl Trait` args) resolves the concrete `Instance` at compile time: rustc monomorphizes one copy per `T`, emits a *direct* call LLVM can inline, zero runtime indirection at the cost of duplicated code. Dynamic dispatch (`&dyn Trait`) represents the value as a *fat pointer* — data pointer + vtable pointer — and lowers each method call to `InstanceKind::Virtual(def_id, idx)` (`ty/instance.rs:117`), a virtual call that loads a function pointer from the vtable at runtime (`rustc_codegen_ssa/src/meth.rs:46`). The vtable is computed by the `vtable_entries` query as a list of `VtblEntry` (`ty/vtable.rs:13`), laid out `[drop, size, align, ...method ptrs...]`. Only *dyn-compatible* (object-safe) traits may become `dyn Trait`, gated by `dyn_compatibility_violations` (`rustc_trait_selection/src/traits/dyn_compatibility.rs`).

## 1. Static dispatch — monomorphization, direct calls

```rust
trait Speak { fn say(&self) -> u8; }
fn generic<T: Speak>(x: &T) -> u8 { x.say() }   // static
fn imp(x: &impl Speak) -> u8 { x.say() }        // static (impl Trait arg == generic)
```

`generic::<Cat>` and `generic::<Dog>` are separate `Instance`s; `x.say()` resolves to a concrete `Instance` (e.g. `InstanceKind::Item(<Cat as Speak>::say)`) at compile time, so codegen emits a direct call to a known symbol LLVM can inline. `impl Trait` *in argument position* is sugar for a generic param — same path. Cost: code duplication per `T`. This is the mono collector's job (see [[18-monomorphization]]).

## 2. Dynamic dispatch — the fat pointer and `Virtual`

`&dyn Speak` is an unsized trait object behind a wide pointer: a data pointer to the erased value + a vtable pointer (its `DynMetadata`; lang item at `ty/sty.rs:1770`). A method call can't be resolved to a concrete function, so rustc models it as `InstanceKind::Virtual(def_id, idx)` (`ty/instance.rs:109`):

> "Calls to `Virtual` instances must be codegen'd as virtual calls through the vtable. This means we might not know exactly what is being called."

In codegen, `codegen_call_terminator` detects a `Virtual` callee on its receiver arg and loads the fn pointer from the vtable by slot index (`rustc_codegen_ssa/src/mir/block.rs:1238`), via `VirtualIndex::get_fn` (`meth.rs:46`). `load_vtable` computes `vtable_byte_offset = idx * ptr_size`, does `inbounds_ptradd` + `load`, marked invariant/nonnull (`meth.rs:126`). That's the one runtime indirection: one load + one indirect call, no inlining across it, but only one copy of the calling code.

## 3. Implementation: how vtables are built and laid out

The shape is `VtblEntry` (`ty/vtable.rs:13`):

```rust
pub enum VtblEntry<'tcx> {
    MetadataDropInPlace,        // header: drop glue
    MetadataSize,               // header: size
    MetadataAlign,              // header: align
    Vacant,                     // excluded method (e.g. Self: Sized)
    Method(Instance<'tcx>),     // dispatchable method pointer
    TraitVPtr(TraitRef<'tcx>),  // supertrait vtable pointer (upcasting)
}
```

The first three are fixed and come first — `COMMON_VTABLE_ENTRIES = [DropInPlace, Size, Align]` at indices `0/1/2` (`vtable.rs:45`). So **layout order = drop glue, size, align, then method pointers** (plus optional supertrait vptrs). Two layers:

- `vtable_entries` (query, `rustc_trait_selection/src/traits/vtable.rs:234`) walks vtable *segments* via `prepare_vtable_segments`: the first pushes `COMMON_VTABLE_ENTRIES` (`:247`); each `TraitOwnEntries` segment resolves a concrete `Instance` per method (`expect_resolve_for_vtable`, `:284`) and pushes `Method(instance)`, or `Vacant` if where-clauses can't hold (`:279`). Supertraits are ordered so upcasting to the first is zero-cost (`:42`).
- `vtable_allocation_provider` (`ty/vtable.rs:84`) turns the list into a constant `Allocation` of `n * ptr_size` bytes: drop glue ptr or null if `!needs_drop` (`:123`), size/align as integers (`:133`), `Vacant` leaves uninit (`:135`), `Method` writes the fn ptr (`:136`), `TraitVPtr` writes the supertrait vtable ptr (`:142`).

Codegen materializes it: `get_vtable` calls `tcx.vtable_allocation(...)`, lowers via `static_addr_of`, memoizes in `cx.vtables()` (`meth.rs:103`).

### Concrete vtable for `&dyn Speak` holding a `Cat`

| slot | `VtblEntry` | contents |
|---|---|---|
| 0 | `MetadataDropInPlace` | ptr to `Cat`'s drop glue, or **null** (Cat needs no drop) |
| 1 | `MetadataSize` | `size_of::<Cat>()` |
| 2 | `MetadataAlign` | `align_of::<Cat>()` |
| 3 | `Method(<Cat as Speak>::say)` | ptr to `Cat::say` |

`obj.say()` → load word 3, indirect-call with the data pointer as `&self`. `Dog`'s vtable has `Dog::say` at word 3.

## 4. Object safety / dyn-compatibility gating

You can only write `dyn Trait` if the trait is *dyn-compatible* (formerly "object safe"). `is_dyn_compatible` = `dyn_compatibility_violations(trait).is_empty()` (`dyn_compatibility.rs:61`). Violations (`rustc_middle/src/traits/mod.rs:760`): `SizedSelf`, `SupertraitSelf`, generic methods / no-`self` methods (`MethodViolation`), associated consts, GATs. Even on a compatible trait, methods with a `Self: Sized` bound are non-dispatchable (`is_vtable_safe_method`, `:69`) and become `Vacant` slots.

## 5. Trade-offs

| | Static (`<T: Trait>` / `impl Trait`) | Dynamic (`dyn Trait`) |
|---|---|---|
| Resolution | compile time, monomorphized `Instance` | runtime, vtable load (`Virtual`) |
| Call form | direct, inlinable | indirect through fn ptr |
| Code size | one copy **per type** (can bloat) | one copy total |
| Speed | fastest; enables inlining/const-prop | extra load + indirect call |
| Flexibility | type fixed per instantiation | heterogeneous (`Vec<Box<dyn Trait>>`) |
| Pointer | thin | fat (data + vtable) |
| Constraint | none beyond the bound | trait must be dyn-compatible |

Mitigation: `-Zvirtual-function-elimination` under fat LTO routes vtable loads through `llvm.type.checked.load` so LLVM can devirtualize/strip dead slots (`meth.rs:136`).

## See also

- [[18-monomorphization]] · [[30-abi-status]] · [[16-type-inference]] · [[07-glossary]]
