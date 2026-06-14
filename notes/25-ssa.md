# SSA (Static Single Assignment)

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_mir_transform/src/ssa.rs`, `compiler/rustc_codegen_ssa`
- **Date:** 2026-06-15

## Question

What is SSA form, why do compilers use it, and how does it show up in rustc — given that MIR is not itself SSA, what does `SsaLocals` compute, and what is `rustc_codegen_ssa` really?

## Short answer

SSA (Static Single Assignment) is an IR property where every variable is assigned exactly once, and control-flow joins are reconciled by φ ("phi") nodes that pick a value depending on which predecessor block was taken. It makes def-use chains explicit, which simplifies optimizations like GVN, constant propagation, and DCE. rustc's MIR is **not** strictly SSA — locals can be reassigned — but `compiler/rustc_mir_transform/src/ssa.rs` provides an `SsaLocals` analysis that identifies the subset of locals that *happen* to behave SSA-like (assigned once, with that definition dominating all uses), and several MIR passes gate their rewrites on it. True SSA-with-phis lives in LLVM IR, not MIR; `rustc_codegen_ssa` is the **backend-agnostic codegen layer**, where rustc decides which MIR locals become SSA values vs needing stack slots (allocas) before lowering to the backend's SSA IR.

## What SSA form is

In SSA, each named value is the target of exactly one assignment. Where a variable would be assigned in multiple places, SSA renames each definition (`x1`, `x2`, …) and, at merges, inserts a **φ node** selecting which definition reaches the join.

```text
// before (not SSA — x reassigned)        // after (SSA)
x = 1                                      bb0: x1 = 1; br cond, bb1, bb2
if cond { x = 2 }                          bb1: x2 = 2; goto bb2
use(x)                                     bb2: x3 = phi [x1 from bb0], [x2 from bb1]
                                                use(x3)
```

The φ `x3 = phi [x1, bb0], [x2, bb1]` means "take `x1` if we came from `bb0`, `x2` if from `bb1`." It's not a runtime call — it's notation for "value depends on the incoming edge," realized later as moves by the register allocator.

## Why compilers use it

Each value has a single definition, so the **def-use relationship is explicit**: from a use you find the one definition; from a definition you find all uses. That makes analyses local and cheap:

- **GVN / CSE** — identical values get identical numbers; redundant recomputation is reused.
- **Constant propagation** — a constant flows directly to every use, no "was it overwritten?" question.
- **DCE** — a definition with zero uses is trivially dead.

In a non-SSA IR you'd need extra dataflow (reaching definitions) to answer "which assignment does this use see?" SSA bakes that into the IR's structure.

## How it relates to rustc: MIR is not SSA, but `SsaLocals` is

MIR locals (`_0`, `_1`, …) are mutable storage slots: a local can be assigned many times, written through a borrow, or have its address taken. So MIR is not SSA.

Instead, `ssa.rs` computes which locals are *de facto* SSA. The module doc states the criteria (`ssa.rs:1`):

> 1/ They are only assigned-to once... 2/ This single assignment dominates all uses.

`SsaLocals` (`ssa.rs:20`) records, per local, a `Set1<DefLocation>` of assignments (`ssa.rs:22`). A local is SSA iff its set is `One` — `is_ssa` (`ssa.rs:111`):

```rust
pub(super) fn is_ssa(&self, local: Local) -> bool {
    matches!(self.assignments[local], Set1::One(_))
}
```

Details verified in source:

- Builds dominators, visits the body in **reverse postorder** (`ssa.rs:63`) so each assignment is seen before its uses; lets it check in one pass that the single def dominates every use.
- `SsaVisitor` demotes a local to `Set1::Many` on any mutating use, raw-borrow, or use not dominated by a def (`ssa.rs:236`, `check_dominates` `:212`).
- **Subtlety** (`ssa.rs:5`): a local only *immutably* borrowed can still be SSA, but only if its type is `Freeze` (no interior mutability) — the `Freeze` check is deferred until after the walk (`ssa.rs:73`).
- Also computes **copy equivalence classes** (`copy_classes`, `compute_copy_classes` `:296`): `_b = _a; _c = _a` unify to one representative. So `SsaLocals` is "SSA-detection plus copy-class info."

`SsaLocals` does **not** rewrite MIR into SSA or insert φ nodes — it's purely analysis enabling other passes. Consumers: `gvn.rs:139`, `copy_prop.rs:33`, `ref_prop.rs`, `ssa_range_prop.rs`, `simplify_comparison_integral.rs`.

(The same file also hosts unrelated helpers `StorageLiveLocals` `:364` and the `MaybeUninitializedLocals` dataflow `:400`.)

## `rustc_codegen_ssa`: the backend-agnostic codegen layer

The name can mislead. Its own doc (`compiler/rustc_codegen_ssa/src/lib.rs:11`):

> This crate contains codegen code that is used by all codegen backends (LLVM and others)...

So it's the **shared, backend-independent half of codegen**; concrete backends (`rustc_codegen_llvm`/`_gcc`/`_cranelift`) implement its traits (see [[23-gcc-relationship]]). The "ssa" reflects that this layer lowers MIR toward an SSA-style IR for the backend (LLVM IR is SSA); it is not LLVM-specific.

The actual SSA decision during codegen lives in `compiler/rustc_codegen_ssa/src/mir/analyze.rs` ("An analysis to determine which locals require allocas and which do not", `:1`). Its `non_ssa_locals` (`:16`) classifies each MIR local: locals whose address is taken, that are reassigned, or otherwise can't live purely as a value become `LocalKind::Memory` (`:47`) and get **stack slots (allocas)**; the rest become plain SSA values in the backend IR. This is rustc's analogue of LLVM's `mem2reg`, done up front. (Independent from `rustc_mir_transform`'s `SsaLocals`; same concept, separate code.)

## Where the φ nodes actually are

MIR has no φ nodes — joins are ordinary writes to mutable locals across basic blocks. φ nodes appear once rustc lowers MIR to **LLVM IR**, which is genuinely SSA: backend-SSA locals become LLVM SSA values, memory locals become allocas, and LLVM (or `mem2reg`) introduces φ nodes at joins. In short: **MIR ≈ "SSA where convenient, tracked by analysis"; LLVM IR = true SSA with phis.**

## See also

- [[20-mir-format]] · [[13-mir-optimizations-catalog]] · [[12-optimization-pipeline]] · [[23-gcc-relationship]] · [[07-glossary]]
