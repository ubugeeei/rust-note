# How Rust's smart pointers are implemented

- **Upstream commit:** `4f8ea98e`
- **Area:** `library/alloc` (boxed/rc/sync), `library/core/src/cell.rs`, `library/core/src/ops/deref.rs`
- **Date:** 2026-06-15

## Question

How does std implement its core smart pointers (`Box`, `Rc`, `Arc`, `Cell`/`RefCell`), and what are their real in-memory representations and mechanics?

## Short answer

Each is a thin wrapper over a raw pointer or `UnsafeCell` plus a little bookkeeping. `Box<T>` is a `Unique<T>` (one word for sized `T`) tied to the compiler via the `owned_box` lang item; it allocates with the global allocator and frees on `Drop`. `Rc<T>`/`Arc<T>` both point to one heap allocation holding *strong*, *weak*, and the value contiguously — `Rc` uses `Cell<usize>` (non-atomic, `!Send`/`!Sync`), `Arc` uses atomics (thread-safe). `Cell`/`RefCell` give interior mutability over `UnsafeCell<T>`, with `RefCell` adding a runtime borrow counter that panics on violation. `*` and method auto-deref are powered by `Deref`/`DerefMut`; cleanup by `Drop`.

## 1. `Box<T>` — unique owning heap pointer

Special-cased via the `owned_box` lang item (`library/alloc/src/boxed.rs:227`):

```rust
#[lang = "owned_box"]
pub struct Box<T: ?Sized, A: Allocator = Global>(Unique<T>, A);
```

A `Box` is a `Unique<T>` + allocator — one word (thin pointer) for sized `T`, a fat pointer for `dyn`/`[T]`. `Box::new` → `box_new_uninit` → `Global.allocate(layout)`, `handle_alloc_error` on failure (`boxed.rs:247`). `Deref` re-borrows through the pointer (`:2247`). `Drop` (`:1944`) does **not** drop the value (the compiler does that first) — it only deallocates (skipping ZST layouts).

## 2. `Rc<T>` — non-atomic refcounting

A `NonNull` to one heap node with two counts + the value (`library/alloc/src/rc.rs:285`):

```rust
struct RcInner<T: ?Sized> { strong: Cell<usize>, weak: Cell<usize>, value: T }
pub struct Rc<T: ?Sized, A = Global> { ptr: NonNull<RcInner<T>>, alloc: A }
```

`Cell<usize>` counts → `Rc` is `!Send`/`!Sync` (`rc.rs:330,338`). `Rc::new` allocates the whole `RcInner` with both counts = 1 (`:426`). `clone` bumps strong + copies the pointer (`:2510`); overflow → `abort()`. `drop` decrements strong, and at zero `drop_slow` runs the value's destructor via `drop_in_place`, freeing storage when the last weak ref drops (`:393`). `Weak<T>` (`:3190`) holds the same pointer but tracks only the weak count (breaks cycles), also `!Send`/`!Sync`.

### Rc heap layout

```
  stack                 heap: one RcInner<T> allocation
 ┌──────────┐          ┌──────────────────────────────┐
 │ Rc<T>    │   ptr    │ strong: Cell<usize> = N        │  # of Rc<T>
 │  .ptr ───┼────────▶ │ weak:   Cell<usize> = M (+1)   │  # of Weak<T> (+1 while any Rc lives)
 │  .alloc  │          │ value:  T          = ...       │  payload, inline
 └──────────┘          └──────────────────────────────┘
   value dropped when strong==0; allocation freed when weak==0.
```

## 3. `Arc<T>` — atomic refcounting

Mirrors `Rc`, counters atomic (`library/alloc/src/sync.rs:386`): `struct ArcInner { strong: Atomic<usize>, weak: Atomic<usize>, data: T }`. `Arc` is `Send`/`Sync` when `T: Send + Sync` (`sync.rs:279`). `clone` uses a `Relaxed` `fetch_add` with an abort past `MAX_REFCOUNT`; `drop` uses `Release` on the decrement + an `Acquire` fence on the final drop, so prior uses *happen-before* destruction across threads (`sync.rs:2830`).

## 4. `Cell<T>` / `RefCell<T>` — interior mutability

Built on `UnsafeCell<T>`, the one primitive the compiler allows for shared mutation — a `#[repr(transparent)]` newtype whose `get` yields `*mut T` (`library/core/src/cell.rs:2321`). `Cell<T>` is just `UnsafeCell<T>` (`!Sync`), offering value-level get/set/replace, no references handed out → no runtime tracking. `RefCell<T>` (`cell.rs:849`) adds `borrow: Cell<BorrowCounter>` (`isize`, `:945`): `0` unused, **positive** = N shared borrows, **negative** = a write borrow. `borrow()`/`borrow_mut()` panic on violation (`:1121`,`:1221`). RAII guards `BorrowRef`/`BorrowRefMut` (`:1556`,`:2044`) bump/restore the counter on construction/`Drop`; they're embedded in `Ref`/`RefMut` returned to callers. `RefCell` is `!Sync`.

## 5. `Deref`/`DerefMut` and `Drop`

`Deref` is the `deref` lang item (`library/core/src/ops/deref.rs:133`); `*p` for a non-reference rewrites to `*Deref::deref(&p)`. The same machinery drives **method-call auto-deref**: to resolve `p.method()`, the compiler repeatedly applies `Deref` (`p`, `*p`, `**p`, …) until it finds the method — which is why `Box`/`Rc`/`Arc` transparently expose `T`'s methods (each provides `Deref` with `Target = T`). They also assert `DerefPure` (stable, side-effect-free). `Drop` handles deallocation/refcount logic; the inner value's destructor runs via the compiler (Box) or `drop_in_place` (Rc/Arc).

## See also

- [[24-libc-dependency]] · [[38-multithreading]] · [[21-arena]] · [[07-glossary]]
