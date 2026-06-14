# Branch optimizations deep dive

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_mir_transform/src/{simplify_branches,simplify_comparison_integral,match_branches,early_otherwise_branch,jump_threading}.rs`
- **Date:** 2026-06-15

## Question

How do the MIR branch optimizations work?

## Short answer

They fold switches/comparisons/branches into simpler forms. From cheap to fancy:
**SimplifyConstCondition** (known-constant switch → goto), **SimplifyComparison
Integral** (`Eq`-then-switch-on-bool → switch on the integer), **MatchBranch
Simplification** (two arms differing only by a const bool → `Eq`/`Ne`; or unify
identical arms), **EarlyOtherwiseBranch** (collapse repeated discriminant reads in
nested matches), and **JumpThreading** (a two-phase condition-graph + block
duplication that turns join-then-switch into straight jumps).

## 1. SimplifyConstCondition (`simplify_branches.rs`)

> "A pass that replaces a branch with a goto when its condition is known."

```text
_1 = const true
switchInt(_1) -> [false: bb2, otherwise: bb3]   =>   goto bb3
```

Runs several times in the pipeline (`AfterInstSimplify`, `AfterConstProp`,
`Final`) to harvest conditions that earlier passes proved constant.

## 2. SimplifyComparisonIntegral (`simplify_comparison_integral.rs`)

> "convert `if` conditions on integrals into switches on the integral."

Removes the intermediate boolean and switches on the integer directly:

```text
_3 = Eq(move _4, const 43i32)               switchInt(_4) -> [43i32: bb3,
switchInt(_3) -> [false: bb2,        =>                       otherwise: bb2]
                  otherwise: bb3]
```

## 3. MatchBranchSimplification (`match_branches.rs`)

> "Unifies all targets into one basic block if each statement can have the same
> statement." Enabled only at `-Zmir-opt-level=2` (it can hurt debuggability).

**Case A — two arms differing only by a const bool → `Eq`/`Ne`** (source example):

```text
bb0: switchInt(move _3) -> [42: bb1, otherwise: bb2]
bb1: _2 = const true;  goto bb3      =>    bb0: _2 = Eq(move _3, const 42)
bb2: _2 = const false; goto bb3                 goto bb3
```

**Case B — all targets perform the same assignment → unify into the source
block**, deleting the switch entirely.

Effect: converts **control dependence into data dependence** (branch → arithmetic),
which flattens the CFG and avoids branch mispredictions.

## 4. EarlyOtherwiseBranch (`early_otherwise_branch.rs`)

Collapses repeated reads of the same discriminant in a nested match (source
example):

```rust
match (x, y) {                     let dx = discriminant(x);
  (Some(_), Some(_)) => 0,         let dy = discriminant(y);
  (None,    None)    => 2,   =>    if dx == dy {
  _                  => 1,             match x { Some(_) => 0, None => 2 }
}                                  } else { 1 }
```

The source even draws the BB diagram: a `switchInt(Q)` in BB1 feeding blocks
(BBC/BBD) that each recompute `discriminant(P)` — the pass merges those reads.

## 5. JumpThreading (`jump_threading.rs`) — the advanced one

> "replace join-then-switch control flow patterns by straight jumps."

```text
   X = 0                          X = 0
------------\     /--------     ------------
   X = 1     X---X Switch(X)  =>     X = 1
------------/     \--------     ------------
```

Two-phase algorithm (inspired by the libfirm thesis):

1. **Condition graph construction (walk CFG backwards).** Denote a replacement
   condition as `place ?= value`. A `switchInt(place) -> [value: target]` records
   a `place == value` condition with a `Goto(target)` action. Conditions in a
   successor `s` are inherited as `Chain(s, c)` (fulfilling `c` in this block
   fulfills `c` in `s`). Statements are walked backwards, transforming the
   condition set to the block entry.
2. **Block duplication (propagate forward, reverse postorder).** For each
   fulfilled condition: `Goto(target)` rewrites the terminator; `Chain(target,
   cond)` **duplicates `target`** into a new block that also fulfills `cond`,
   memoized via `duplicate[(target, cond)]` to avoid re-cloning.

Safety/limits:
- **Never thread through a loop header** — would create irreducible control flow.
- **Bounded by `MAX_COST`** — duplication can blow up MIR size.

## Two cross-cutting themes

- **UB preservation**: deleting a `switchInt` deletes the *read* of its operand,
  which can remove UB and change which programs compile — so these are careful on
  pre-runtime MIR (same idea as `preserve_switch_reads` in [[14-cfg-simplification]]).
- **Branch → arithmetic**: passes 2 and 3 trade control dependence for data
  dependence, flattening the CFG for both later passes and the CPU.

## See also

- [[14-cfg-simplification]] — the structural CFG cleanup these interleave with
- [[13-mir-optimizations-catalog]] · [[12-optimization-pipeline]]
