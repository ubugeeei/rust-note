# Lifetimes: how they're implemented (region inference & NLL)

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_infer` (regions), `rustc_borrowck` (NLL, region_infer, polonius)
- **Date:** 2026-06-15

## Question

How are lifetimes implemented?

## Short answer

Lifetimes don't exist at runtime — they're a **compile-time constraint system**.
Internally a lifetime is a **region** (`RegionVid`), and inference works like type
inference: variables + `'a: 'b` ("outlives") constraints solved to a fixpoint. The
modern engine is **NLL (Non-Lexical Lifetimes)** in `rustc_borrowck`, which models
a region as **the set of MIR CFG points where a reference must stay valid**,
derived from **liveness** (where it's actually used) rather than lexical scope.
**Polonius** is a more precise datalog-based successor.

## Regions as inference variables

A lifetime is a region, represented by `RegionVid`, a union-find key
(`infer/unify_key.rs:27`, `RegionVidKey`) — the lifetime analogue of `TyVid` in
[[16-type-inference]].

## Two generations of region solving

- **Lexical region resolution** (older): `infer/lexical_region_resolve` +
  `infer/region_constraints` — collect `'a: 'b` outlives constraints and solve by
  fixpoint, with regions tied to lexical scopes.
- **NLL (Non-Lexical Lifetimes)** (current): in `rustc_borrowck`, regions are
  based on **liveness over the MIR CFG**, not lexical scopes.

## NLL flow (`rustc_borrowck/src/nll.rs` — "entry point of the NLL borrow checker")

```text
1. universal_regions.rs: collect named lifetimes from the fn signature
   ('a, 'static, ...).
2. Each region becomes a region variable (RegionVid). Each variable stands for
   "the set of MIR CFG points where this region is live".
3. constraints/: generate outlives constraints from
      - subtyping while type-checking the MIR,
      - liveness of references,
      - relations among universal regions.
   `'a: 'b` means "'a's point-set must contain 'b's point-set".
4. region_infer/: compute the minimal point-set for each region variable by
   propagating constraints to a fixpoint (like a reachability computation).
5. With regions known, borrow-check: ensure no borrow is used while invalidated
   (conflicting loans).
```

## Why "non-lexical"

Old model: a lifetime lasted to the end of its lexical scope. NLL computes it to
**the last point the reference is actually used** (from liveness), so this
compiles:

```rust
let mut v = vec![1];
let r = &v[0];
println!("{r}");   // last use of r here -> its region ends here
v.push(2);         // OK now (the old lexical model rejected this)
```

## Polonius (next-gen)

`rustc_borrowck/src/polonius/` — a **datalog-based** reformulation tracking
`origin`s and `loan`s for more precise borrow analysis; the in-progress successor
to NLL.

## Mental model

Same machinery as type inference, applied to regions: **variables + constraints +
fixpoint**. The twist is that NLL's "value" for a region is a **set of CFG
points** (a liveness range), which is what makes Rust's borrow checking flow-
sensitive and non-lexical.

## Notes & open questions

- How exactly liveness is computed for regions (`rustc_mir_dataflow`?) and fed
  into constraints.
- NLL vs Polonius: what cases Polonius accepts that NLL rejects.

## See also

- [[16-type-inference]] — the type-variable analogue
- [[08-hir-and-mir]] — borrow check runs on MIR
- borrow check / NLL guide: https://rustc-dev-guide.rust-lang.org/borrow_check.html
