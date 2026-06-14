# How closures are implemented

- **Upstream commit:** `4f8ea98e`
- **Area:** `library/core/src/ops/function.rs`, `compiler/rustc_hir_typeck` (upvar/capture)
- **Date:** 2026-06-15

## Question

How does rustc implement closures — the `Fn`/`FnMut`/`FnOnce` traits, how a closure becomes a struct + trait impl, and how capture modes (incl. Rust 2021 disjoint captures) work?

## Short answer

A closure compiles to an anonymous, compiler-generated struct whose fields are the captured upvars, plus an auto-generated impl of one of the `Fn*` traits whose method body *is* the closure body. The traits form a subtrait chain `Fn: FnMut: FnOnce` (`library/core/src/ops/function.rs:76,163,242`), distinguished by receiver: `call(&self)`, `call_mut(&mut self)`, `call_once(self)`. Which trait a closure implements is *inferred* from how its body uses captures, computed in `compiler/rustc_hir_typeck/src/upvar.rs` (`process_collected_capture_information`, `:645`). Capture *mode* (by-ref/by-mut/by-move) and, since Rust 2021, *granularity* (whole var vs individual fields) are decided there, gated on the edition.

## 1. The Fn / FnMut / FnOnce hierarchy

All in `library/core/src/ops/function.rs`, all lang items:
- `FnOnce<Args>` (`:242`, lang `"fn_once"`) — base; owns `type Output` (`:246`) and `extern "rust-call" fn call_once(self, args)` (`:250`). `self` by value → may consume captures → callable once.
- `FnMut<Args>: FnOnce` (`:163`) — adds `fn call_mut(&mut self, args)` (`:166`); `&mut self` mutates captured state, repeatedly.
- `Fn<Args>: FnMut` (`:76`) — adds `fn call(&self, args)` (`:79`); `&self`, callable repeatedly without mutation.

The supertrait chain means an `Fn` is usable where `FnMut`/`FnOnce` is expected. The call methods use the `extern "rust-call"` ABI (the tuple `Args` is *untupled* at the ABI layer, `rustc_ty_utils/src/abi.rs:577`), enabling `Fn(A, B) -> R` paren-sugar (`#[rustc_paren_sugar]`, `:58`). The module also blanket-impls for references (`&F: Fn` when `F: Fn`, `:258`).

## 2. Desugaring: closure → anonymous struct + impl

```rust
fn make_adder(x: i32) -> impl Fn(i32) -> i32 { move |y| x + y }
```

becomes, conceptually:

```rust
struct Closure_make_adder { x: i32 }   // captured env = fields
impl FnOnce<(i32,)> for Closure_make_adder {
    type Output = i32;
    extern "rust-call" fn call_once(self, (y,): (i32,)) -> i32 { self.x + y }
}
impl FnMut<(i32,)> for Closure_make_adder { /* call_mut(&mut self, ...) */ }
impl Fn<(i32,)>   for Closure_make_adder { /* call(&self, ...) */ }
```

Fields = captured upvars; method args = closure params (tupled). The chosen `ClosureKind` (top trait — here `Fn`) is inferred in typeck. In MIR, rustc lowers the body once for the inferred kind and generates *shims* for the others: `ClosureOnceShim` (`rustc_mir_transform/src/shim.rs:65`) makes `call_once` forward to `call_mut` after a `RefMut` adjustment.

## 3. Capture modes & Rust 2021 disjoint captures

Capture analysis is the heart of `rustc_hir_typeck/src/upvar.rs`. For each closure (`analyze_closure`, `:165`), an `ExprUseVisitor` records the minimal borrow kind per accessed `Place`, escalating along the lattice `ImmBorrow → UniqueImmBorrow → MutBorrow` (`:1`). The capture *clause* then adjusts each place (`:698`): `CaptureBy::Ref` → by shared/unique/mut ref; `CaptureBy::Value` (`move`) → by value; `CaptureBy::Use` → the `use(...)` precise-capture form. The closure *kind* is inferred in `process_collected_capture_information` (`:645`): starts at `LATTICE_BOTTOM` and escalates to `FnOnce` when a capture is consumed by value (`:671`).

**Disjoint captures (RFC 2229, edition-gated).** Pre-2021, a closure captured whole *variables*; in 2021+ it captures the minimal set of *places* (fields/paths) via `compute_min_captures` (`:791`). The gate is `enable_precise_capture(span) = span.at_least_rust_2021()` (`:2683`). When false (≤2018), `analyze_closure` coarsens back to whole-variable capture and clears projections (`:340`). Using the *span* (not a global flag) means macro-generated closures follow the macro's edition. The `RUST_2021_INCOMPATIBLE_CLOSURE_CAPTURES` lint (`:1193`) warns where the change alters drop order or auto-traits. So in 2021, `|| println!("{}", point.x)` captures only `point.x`, leaving `point.y` free.

## 4. Move semantics; `impl Fn` vs `dyn Fn`

If the body moves a captured non-`Copy` value out, the kind escalates to `FnOnce` (the doc example at `function.rs:202`). Passing closures: **`impl Fn`** = static dispatch (concrete type, inlinable; [[18-monomorphization]]); **`dyn Fn`** = dynamic dispatch via a fat pointer + vtable ([[32-static-dynamic-dispatch]]). Since the struct's size depends on captures, only `impl Fn`/generics return a closure by value; heterogeneous closures go behind `Box<dyn Fn...>`.

## 5. Coroutines share the machinery

Generators/`async` blocks reuse the same upvar pipeline: `analyze_closure` handles `Closure`, `CoroutineClosure`, and `Coroutine` uniformly (`:176`). `rustc_middle/src/ty/closure.rs` maps a coroutine-closure's captures onto the child coroutine; the shim infra also covers `ConstructCoroutineInClosureShim` (`shim.rs:77`).

## See also

- [[32-static-dynamic-dispatch]] · [[18-monomorphization]] · [[22-ast-lowering]] · [[11-editions]] · [[07-glossary]]
