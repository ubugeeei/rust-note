# Pattern analysis (exhaustiveness & usefulness)

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_pattern_analysis`
- **Date:** 2026-06-15

## Question

How does rustc decide whether a `match` is exhaustive and whether each arm is reachable, and how is that logic packaged so it can be reused outside rustc (e.g. rust-analyzer)?

## Short answer

`rustc_pattern_analysis` implements one algorithm — *usefulness* — that answers both questions. A pattern `q` is "useful" relative to the patterns above it if some value matches `q` but none of them; an arm is redundant iff it is not useful, and a `match` is exhaustive iff the wildcard `_` is not useful (`usefulness.rs:37`). The algorithm decomposes values into a `Constructor` plus fields, arranges arm patterns into a `Matrix`, and repeatedly *specializes* the matrix by each constructor, generating `WitnessPat`s for values that escape coverage (`usefulness.rs:110`). It's generic over a `PatCx` trait (`lib.rs:36`) so the engine is type-system-agnostic; rustc plugs in via `RustcPatCtxt` (`rustc.rs:79`), and `rustc_mir_build`'s THIR match check calls `analyze_match` (`check_match.rs:407`).

## The problem it solves

The crate's summary (`usefulness.rs:17`): given one pattern per arm, compute (a) values matched by no arm (non-exhaustiveness witnesses), and (b) per subpattern, whether removing it changes anything (reachability). These feed: **non-exhaustive match** `E0004` (`check_match.rs:550`), **unreachable/redundant arm** lints (`:542`), and **`if let` refutability** (irrefutable iff witness list empty, `:660`). Computing exhaustiveness is NP-complete (SAT reduces to it, `usefulness.rs:13`).

## Core concepts

- **Constructors** (`constructor.rs:687`): every value is a `Constructor` + fields. Real: `Struct`, `Variant(idx)`, `Ref`, `Slice`, `Bool`, `IntRange`, `Str`, float ranges. Synthetic/pattern-only: `Wildcard`, `Or`, `Missing` ("all constructors not seen", used for diagnostics, `:53`), `NonExhaustive` (un-listable, e.g. `f64`/`#[non_exhaustive]`), `Hidden`, `Never`, `PrivateUninhabited`.
- **`DeconstructedPat`** (`pat.rs:32`): a pattern lowered to `ctor` + `fields` + `arity` + `ty`, with a `uid: PatId` for per-subpattern lookup. `WitnessPat` (`pat.rs:251`) is the dual the algorithm *builds up* to describe a missing value.
- **Specialization** (`usefulness.rs:110`): `specialize(c, p)` strips one layer of constructor `c` off `p`, returning field sub-patterns, or nothing if incompatible (`Constructor::is_covered_by`, `constructor.rs:811`). Done at three levels: pattern, row (`PatStack::pop_head_constructor`, `:1068`), matrix (`Matrix::specialize_constructor`, `:1283`).
- **The matrix** (`usefulness.rs:1218`): one `MatrixRow` per arm + `place_info` (one per column = one scrutinee place); tracks the virtual `_` row for exhaustiveness.
- **Constructor splitting** (`ConstructorSet::split`, `constructor.rs:1046`): groups identically-behaving constructors (can't list every `u64`); the `ConstructorSet` for a type comes from `PatCx::ctors_for_ty` (`lib.rs:72`), split into present vs missing.

## The one-pass algorithm

`compute_match_usefulness` (`usefulness.rs:1835`) builds the matrix and calls `compute_exhaustiveness_and_usefulness` (`:1704`):
1. **Base case — no columns** (`:1717`): a row is useful iff no unguarded row sits above it; a surviving virtual wildcard means non-exhaustive.
2. **Inductive case**: split the head column's constructors; for each, specialize, recurse, then map witnesses back up with `apply_constructor` (the inverse of specialization, `:1754`).

It loops "for each constructor, for each pattern", computing arm usefulness *and* exhaustiveness in a single traversal. Output `UsefulnessReport` (`:1822`): `arm_usefulness`, `non_exhaustiveness_witnesses`, `arm_intersections`.

## Generic over `PatCx` (reuse)

The crate is parameterized on `PatCx` (`lib.rs:36`), abstracting all type-system specifics: assoc types `Ty`, `VariantIdx`, `ArmData`, `PatData`, `Error`, and ops `ctor_arity`/`ctor_sub_tys`/`ctors_for_ty` (`lib.rs:60`). The pure algorithm never mentions rustc types; rustc-only code is behind `#[cfg(feature = "rustc")]`. rustc's impl is `RustcPatCtxt` (`rustc.rs:79`), lowering THIR `Pat`s via `lower_pat` (`rustc.rs:461`). rust-analyzer supplies its own `PatCx`.

## Where rustc calls it

THIR match checking: `rustc_mir_build/src/thir/pattern/check_match.rs`. Query entry `check_match` (`:32`) → `analyze_patterns` → `rustc_pattern_analysis::rustc::analyze_match` (`:407`). `analyze_match` (`rustc.rs:1070`) reveals the scrutinee type, derives `PlaceValidity`, calls `compute_match_usefulness`, and runs the `non_exhaustive_omitted_patterns` lint when already exhaustive. The report drives unreachable-subpattern lints (`:412`), arm reachability (`:542`), and the `E0004` error from witnesses (`:550`).

## Small example: `match x: Option<bool>` missing an arm

```rust
match x {
    None => {}
    Some(true) => {}
    // missing: Some(false)
}
```

Head type `Option<bool>`; `ctors_for_ty` yields `Variants { None, Some }`, both present. Specializing by `Some` peels its `bool` field, whose set is `{ false, true }`; only `true` appears → `false` missing. The base case fires for the surviving wildcard, and `apply_constructor` rebuilds: `false` → `Some(false)`. Result: a single `WitnessPat` `Some(false)`, formatted as:

```
error[E0004]: non-exhaustive patterns: `Some(false)` not covered
```

## See also

- [[08-hir-and-mir]] · [[20-mir-format]] · [[29-type-ir]] · [[07-glossary]]
