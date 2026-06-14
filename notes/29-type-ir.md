# `rustc_type_ir`: the type-system IR abstraction

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_type_ir`, `compiler/rustc_middle/src/ty`
- **Date:** 2026-06-15

## Question

What is `rustc_type_ir`, why is it a crate separate from `rustc_middle::ty`, and how do its generic `TyKind<I>` / `ConstKind<I>` / `RegionKind<I>` relate to rustc's concrete `ty::TyKind<'tcx>`?

## Short answer

`rustc_type_ir` is a *backend-agnostic* definition of the type-system IR: the shapes of types, consts, regions, predicates, binders, and inference variables, parameterized over an `Interner` trait rather than over `TyCtxt`. Concrete data (a `DefId`, an interned `Ty`) is supplied through `Interner`'s associated types, so the same enums can be instantiated by rustc (`TyCtxt<'tcx>`), shared by the next-gen trait solver (`rustc_next_trait_solver`), and reused by rust-analyzer. The generic enums live here (`TyKind<I>` `ty_kind.rs:98`, `ConstKind<I>` `const_kind.rs:21`, `RegionKind<I>` `region_kind.rs:138`); rustc's `ty::TyKind<'tcx>` is just `ir::TyKind<TyCtxt<'tcx>>` (`sty.rs:37`). **`rustc_type_ir` holds the abstract definitions; `rustc_middle::ty` is rustc's concrete instantiation via `impl Interner for TyCtxt`.**

## Why a separate crate: the `Interner` trait

The whole reason is to factor the type-system data structures away from rustc's interner. `Interner` is the seam (`interner.rs:22`) — a large bag of associated types naming everything a type-system IR needs but can't define structurally: `type Ty: Ty<Self>` (`:152`), consts (`:181`), regions (`:190`), predicates/clauses (`:202`), generic args (`:94`), and a family of `DefId`-like ids (`:43`).

A telling note above the id list (`interner.rs:45`): rustc defines `TraitId`, `FunctionId`, etc. all as plain `DefId`, but the abstraction keeps them distinct *"because rust-analyzer uses different types so this is convenient for it."* The `nightly` Cargo feature (`Cargo.toml:32`, default on) gates rustc-only derives; `rustc_next_trait_solver` depends with `default-features = false` (`rustc_next_trait_solver/Cargo.toml:12`).

## The generic kind enums

`TyKind<I>` (`ty_kind.rs:98`) threads every rustc-specific payload through `I::...`:

```rust
pub enum TyKind<I: Interner> {
    Bool, Char, Int(IntTy), Uint(UintTy), Float(FloatTy),
    Adt(I::AdtDef, I::GenericArgs),         // :122
    Ref(I::Region, I::Ty, Mutability),      // :150
    FnPtr(ty::Binder<I, FnSigTys<I>>, FnHeader<I>),  // :184
    Alias(AliasTy<I>),                      // :253
    Param(I::ParamTy),                      // :256
    Bound(BoundVarIndexKind, ty::BoundTy<I>),// :274
    Infer(InferTy),                         // :292
    Error(I::ErrorGuaranteed),              // :297
    // ...
}
```

The variant set is the one every rustc dev knows; only the *contents* are abstracted. `ConstKind<I>` (`const_kind.rs:21`) mirrors this — `Param`, `Infer(InferConst)`, `Bound`, `Unevaluated`, `Value(I::ValueConst)`, `Error`, `Expr`. `RegionKind<I>` (`region_kind.rs:138`) does regions — `ReEarlyParam`, `ReBound`, `ReLateParam`, `ReStatic`, `ReVar`, `RePlaceholder`, `ReErased`, `ReError` (the canonical early/late-bound explanation is at `:147`).

## `InferTy` and friends

Inference variables are pure indices, not parameterized (`ty_kind.rs:608`):

```rust
pub enum InferTy { TyVar(TyVid), IntVar(IntVid), FloatVar(FloatVid), FreshTy(u32), ... }
```

`IntVid`/`FloatVid` get `UnifyKey`/`UnifyValue` impls (`:639`, `:658`) so they drive union-find during unification (see [[16-type-inference]]). `Fresh*` are placeholders from the `TypeFreshener` for caching. `ConstKind` carries `InferConst`; `RegionKind` carries `ReVar(RegionVid)`.

## De Bruijn indices and binders

Higher-ranked types use De Bruijn indexing: `DebruijnIndex` (`lib.rs:89`, `INNERMOST = 0`) with `shifted_in`/`shifted_out` (`lib.rs:152`). Bound vars in the kind enums carry a `BoundVarIndexKind`. The binder is generic (`binder.rs:31`): `Binder<I, T> { value: T, bound_vars: I::BoundVarKinds }`. Universes scope placeholders under `for<..>`: `UniverseIndex` (`lib.rs:303`). `Variance` and `xform` also live here (`lib.rs:225`).

## Relationship to `rustc_middle::ty`

`rustc_middle::ty` re-exports the generic types re-parameterized at `I = TyCtxt<'tcx>` (`sty.rs:35`):

```rust
pub type TyKind<'tcx> = ir::TyKind<TyCtxt<'tcx>>;       // sty.rs:37
pub type Binder<'tcx, T> = ir::Binder<TyCtxt<'tcx>, T>; // sty.rs:43
```

So the `ty::TyKind<'tcx>` rustc matches on daily *is* the generic enum, resolved by `impl<'tcx> Interner for TyCtxt<'tcx>` (`context/impl_interner.rs:29`), which fills the associated types (`type DefId = DefId`, `type GenericArgs = ty::GenericArgsRef<'tcx>`, etc.).

The interned value users pass around is `Ty<'tcx>` (`ty/mod.rs:514`):

```rust
pub struct Ty<'tcx>(Interned<'tcx, WithCachedTypeInfo<TyKind<'tcx>>>);
```

an arena-interned, hash-consed pointer to a `TyKind<'tcx>` + cached flags (see [[21-arena]]). The bridge back to the structural enum is the `IntoKind` trait (`inherent.rs:589`, `fn kind(self) -> Self::Kind`), required of the `Interner`'s `Ty` type (`inherent.rs:28`). That's why generic IR code can write `t.kind()` regardless of physical storage.

## Sharing with the next-gen trait solver

`rustc_next_trait_solver` is written entirely against `Interner` (not `TyCtxt`), depending on `rustc_type_ir` with `default-features = false`. The payoff: the new solver's logic is generic over any interner — runs inside rustc (`I = TyCtxt`) and, in principle, rust-analyzer. `Interner` even exposes `next_trait_solver_globally` (`interner.rs:39`). Shared scaffolding (`search_graph`, `solve`) are modules of this crate (`lib.rs:33`).

## See also

- [[16-type-inference]] · [[17-lifetimes]] · [[21-arena]] · [[07-glossary]]
