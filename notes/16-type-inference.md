# Type inference: how it's implemented

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_infer`, `rustc_hir_typeck`, `rustc_trait_selection`
- **Date:** 2026-06-15

## Question

How is type inference implemented?

## Short answer

Unknown types are **inference variables** (`Ty::Infer(InferTy)`), managed by a
**union-find** (`UnificationTable`). Type checking (`rustc_hir_typeck`) walks the
HIR, creates fresh variables for unknowns, and emits **equality/subtyping
constraints** that `rustc_infer` resolves by unifying variables or pinning them to
concrete types. Trait obligations accumulate in a `FulfillmentContext` solved by
the trait solver. Unresolved integer/float vars **fall back** to `i32`/`f64`;
anything still unresolved is a "type annotations needed" error. The whole state is
**snapshot/rollback**-able to try alternatives (coercions, method probing).

## Three crates

- **`rustc_infer`** — the inference engine ("defines the type inference engine"):
  low-level equality & subtyping.
- **`rustc_hir_typeck`** — the per-function-body type-check pass; walks HIR and
  generates constraints.
- **`rustc_trait_selection`** — trait solving (obligation fulfillment).

## Core: inference variables + union-find

Unknown types are `Ty::Infer(InferTy)` (`rustc_type_ir/src/ty_kind.rs:608`):

```rust
pub enum InferTy {
    TyVar(TyVid),      // general type variable, e.g. `_`
    IntVar(IntVid),    // `{integer}` — an unresolved integer literal type
    FloatVar(FloatVid),// `{float}`
    // (plus Fresh* variants used during canonicalization)
}
```

Managed by union-find tables (the `ena` crate via
`rustc_data_structures::unify`), in `infer/type_variable.rs:66,85`:

```rust
eq_relations:          UnificationTableStorage<TyVidEqKey>,  // equality
sub_unification_table: UnificationTableStorage<TyVidSubKey>, // subtyping
```

- Unifying two type vars **merges** them in the union-find.
- Resolving a var to a concrete type records that as the group's value.

## The inference flow

```text
1. Create an InferCtxt (inference context).
2. Walk HIR exprs (FnCtxt::check_expr); make a fresh TyVar per unknown.
3. Emit equality/subtyping constraints from each expr; unify updates the tables:
      let x = 1;        -> x : IntVar(?i)
      let y: u8 = x;    -> unify ?i = u8
4. Trait obligations go into a FulfillmentContext (traits/fulfill.rs:60),
   solved by SelectionContext / the next-gen solver (rustc_next_trait_solver).
   Solving associated types etc. resolves more variables.
5. Int/float fallback: leftover {integer} -> i32, {float} -> f64.
6. resolve (infer/resolve.rs) substitutes inference vars with final types;
   any still-unresolved var -> "type annotations needed".
```

## Snapshot & rollback

Inference state is journaled with `UndoLog` and supports **snapshot/rollback**
(`infer/snapshot`). This powers "try, and undo if it fails" — method probing,
coercion attempts, and speculative trait selection.

## Mental model

Type inference = **constraint solving by union-find**: variables for the
unknowns, equality/subtyping constraints from the program, trait solving to
discharge bounds and pin associated types, fallback for ambiguity, and a
rollback-able transaction log to explore choices.

## Notes & open questions

- Subtyping vs equality: why two unification tables? (`eq` vs `sub`) — worth a
  follow-up on how coercion/variance interacts with `sub_unification_table`.
- The trait solver itself (`rustc_next_trait_solver`) deserves its own note.
- Region/lifetime inference reuses the same variable+constraint idea — see
  [[17-lifetimes]].

## See also

- [[08-hir-and-mir]] — typeck runs on HIR
- [[17-lifetimes]] — region inference, the lifetime analogue
- type inference guide: https://rustc-dev-guide.rust-lang.org/type-inference.html
