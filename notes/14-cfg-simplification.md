# CFG simplification (`SimplifyCfg`) deep dive

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_mir_transform/src/simplify.rs`
- **Date:** 2026-06-15

## Question

How does `SimplifyCfg` actually work?

## Short answer

`SimplifyCfg` removes unnecessary basic blocks: it **collapses goto chains**,
**merges a block into its single goto-predecessor**, turns a **`SwitchInt` whose
successors are all identical into a `Goto`**, strips `Nop`s, and removes dead
blocks. The engine is `CfgSimplifier`, which maintains a **`pred_count`**
(predecessors per block) and loops to a fixpoint. It's deliberately **cheap**, so
it runs after every CFG-changing pass — but because it also runs on built/analysis
MIR, it must carefully **preserve UB** (hence `preserve_switch_reads`).

## The cast of transformations

### 1. Strip nops (`strip_nops`)
Remove `StatementKind::Nop` from every block first.

### 2. Collapse goto chains (`collapse_goto_chain`)
A "simple goto" block = **empty statements + terminator is `Goto`**
(`take_terminator_if_simple_goto`, `simplify.rs:212`). Walk the chain and
redirect the start straight to the final target:

```text
bb0: goto bb1
bb1: goto bb2     =>   bb0: goto bb3   (bb1, bb2 become unreferenced)
bb2: goto bb3
bb3: ...
```

`pred_count` is fixed up as edges are rerouted; if the chain is "trivial" (each
link has exactly one predecessor) debuginfo is moved to the final block.

### 3. Merge a block into its sole predecessor (`merge_successor`, `:279`)

If a terminator is `Goto { target }` and `pred_count[target] == 1`, splice the
target's statements + terminator into the current block:

```text
bb0: _1 = ...; goto bb1          bb0: _1 = ...
bb1: _2 = ...; return       =>         _2 = ...
     (pred_count[bb1] == 1)            return
```

### 4. Fold a degenerate switch into a goto (`simplify_branch`, `:306`)

If a `SwitchInt` has **all successors identical**, it's just a `Goto`:

```text
switch(_1) { 0 => bb5, 1 => bb5, otherwise => bb5 }   =>   goto bb5
```

**UB guard:** this is gated by `preserve_switch_reads`. On `MirPhase::Built` /
`Analysis`, or with `-Zmir-preserve-ub`, it is **skipped**, because removing the
`SwitchInt` deletes the *read* of `_1`, and that read might be the thing that
causes UB — dropping it could change which programs compile (an "insta-stability"
hazard). See the WARNING in the module doc.

### 5. Dedup duplicate switch targets (`simplify_duplicate_switch_targets`, `:337`)

A switch arm that points to the same block as `otherwise` is redundant:

```text
switch(_1) { 5 => bbX, otherwise => bbX }   =>   switch(_1) { otherwise => bbX }
```

### 6. Remove dead blocks (`remove_dead_blocks`, `:349`)

Compute reachability (`reachable_as_bitset`) and delete unreachable blocks; also
dedup empty-unreachable blocks (but never confuse cleanup vs non-cleanup blocks).

## The engine: `CfgSimplifier` (`:106`)

```text
new():     compute pred_count via preorder traversal (dead blocks excluded;
           START_BLOCK seeded to 1). Decide preserve_switch_reads from phase.

simplify():                         // returns "did I change anything?"
  strip_nops()
  loop {
    changed = false
    for bb in blocks where pred_count[bb] != 0 {
      take terminator
      for each successor: collapse_goto_chain(successor)
      repeat until stable {
        changed |= simplify_branch(term)     // switch-all-equal -> goto
        changed |= merge_successor(term)      // pull in single-pred goto target
      }
      append merged blocks' statements into bb
      put terminator back
    }
    if !changed { break }
  }
```

Key design points:
- **`pred_count` is the load-bearing data structure.** Every rerouting carefully
  increments/decrements it so "single predecessor" tests stay correct. A block
  reaching `pred_count == 0` is effectively dead and skipped.
- **Fixpoint loop**: collapsing/merging exposes new opportunities, so it repeats
  until a pass makes no change.
- **Caches**: `simplify` returns a bool; the caller must invalidate the body's
  CFG caches (predecessors/dominators) **only if** something changed
  (`simplify_cfg`, `:79`).
- It uses `as_mut_preserves_cfg()` while working, then signals change explicitly.

## Why two passes, run at different cadences

From the module doc:
- **`SimplifyCfg` is cheap** → run it **after every pass that significantly
  changes the CFG**, and **before analysis passes** (it removes dead blocks,
  some of which can be ill-typed).
- **`SimplifyLocals` is expensive** → run **once, right before codegen**.

The ill-typed-dead-block example from the doc:

```rust
fn example() { let _a: char = { return; }; }
// the `{ return; }` block has type `char`, and naive MIR still writes `_a = ()`
// in the unreachable block after `return`. Removing dead blocks avoids the issue.
```

## Critical caveat: this can run on pre-runtime MIR

Because `SimplifyCfg` runs on **built and analysis MIR** (not just optimized
MIR), its effects can influence type-checking / borrow-checking. So it may only
apply transforms that **preserve UB and all non-determinism** — the usual "a
program with UB may do anything" license does **not** apply here. This is exactly
why `simplify_branch` is gated behind `preserve_switch_reads`.

## See also

- [[13-mir-optimizations-catalog]] — where SimplifyCfg sits among the passes
- [[15-branch-optimizations]] — JumpThreading / EarlyOtherwiseBranch / etc.
- [[12-optimization-pipeline]]
