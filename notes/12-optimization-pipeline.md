# Optimization: MIR passes & where LLVM is invoked

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_mir_transform`, `rustc_monomorphize`, `rustc_codegen_ssa`, `rustc_codegen_llvm`
- **Date:** 2026-06-14

## Question

What compiler optimizations happen across AST → HIR → MIR → code, and at what
stage is LLVM invoked and what does it do?

## Short answer

No optimization happens at AST→HIR (that's desugaring) or HIR→MIR build (that's
construction + borrowck). The rustc-level optimizer is the **MIR pass pipeline**
in `rustc_mir_transform` (`run_optimization_passes`, ~40 passes) — language-aware
and relatively light; it shapes/shrinks the IR. After monomorphization, MIR is
lowered to LLVM IR per codegen-unit, and the **heavy optimization is delegated to
LLVM**, invoked from `rustc_codegen_llvm/src/back/write.rs::optimize` →
`llvm_optimize` → `LLVMRustOptimize` (the LLVM PassBuilder pipeline at the chosen
`-O` level), followed optionally by **LTO** (`back/lto.rs`).

## Stage-by-stage

| Stage | Crate | Optimization? |
|-------|-------|---------------|
| AST → HIR | `rustc_ast_lowering` | **None.** Desugaring/normalization only. |
| HIR → MIR build | `rustc_mir_build` | Construction only; then borrowck, drop elaboration, const-checks on *unoptimized* MIR. |
| **MIR optimization** | `rustc_mir_transform` | rustc's own, language-aware optimizer. |
| monomorphization | `rustc_monomorphize` | Instantiate generics to concrete types; partition into codegen units (CGUs). |
| MIR → LLVM IR | `rustc_codegen_ssa` / `_llvm` | Lower to LLVM IR. |
| **LLVM optimization** | LLVM (invoked from `back/write.rs`) | The heavy, target-aware optimizer. |
| LTO + codegen | `back/lto.rs` + LLVM | Link-time opt + machine code. |

## 1. MIR optimization (rustc's own)

`compiler/rustc_mir_transform/src/lib.rs:700` — `run_optimization_passes` lists
~40 passes run in order. Representative ones (in order):

- **UB checks inserted first**: `CheckAlignment`, `CheckNull`, `CheckEnums` —
  on purpose, *before* optimizations can delete the UB (per the source comment).
- **Inlining**: `ForceInline` → `Inline` (MIR-level inlining).
- **Cleanup**: `RemoveStorageMarkers`, `RemoveZsts` (zero-sized types),
  `RemoveUnneededDrops`.
- **CFG simplification**: `UnreachableEnumBranching`, `UnreachablePropagation`,
  `SimplifyCfg`.
- **Value propagation / redundancy**: `ReferencePropagation`,
  `ScalarReplacementOfAggregates` (SROA), **`GVN`** (global value numbering =
  CSE + constant folding), `DataflowConstProp` (constant propagation).
- **Branch opts**: `JumpThreading`, `EarlyOtherwiseBranch`,
  `MatchBranchSimplification`, `SimplifyComparisonIntegral`.
- **Assignment opts**: `CopyProp`, `DeadStoreElimination`,
  `DestinationPropagation`.
- **Layout**: `EnumSizeOpt`.

Mechanics: passes wrapped in `o1(...)` = `WithMinOptLevel(1, ...)` only run at
`-C opt-level >= 1`. Functions marked e.g. `#[optimize(none)]` get
`Optimizations::Suppressed`.

**Why have MIR opts at all if LLVM does the heavy lifting?**
1. **Language-aware**: can use info lost in LLVM IR (drops, ZSTs, enums, panics).
2. **Shrinks IR handed to LLVM** → faster compile + better LLVM output.
3. **Benefits all backends** (Cranelift, GCC), not just LLVM.
4. **Const-eval (CTFE)** also runs on MIR.

## 2. Invoking LLVM and what it does

After lowering MIR → LLVM IR, optimization is kicked off in
`compiler/rustc_codegen_llvm/src/back/write.rs`:

```
optimize (:880)
  -> choose opt_stage from LTO config (:902):
       Fat LTO  -> PreLinkFatLTO
       Thin LTO -> PreLinkThinLTO
       none     -> PreLinkNoLTO
  -> llvm_optimize (:552)
       -> LLVMRustOptimize (:767)   // runs LLVM's PassBuilder pipeline
```

- Opt-level mapping: `to_pass_builder_opt_level` (`:148`) /
  `to_llvm_opt_settings` (`:136`) translate rustc's `-C opt-level=0/1/2/3/s/z`
  into LLVM's `O0/O1/O2/O3/Os/Oz` pipelines.
- LLVM does the **heavy, low-level work**: function inlining, loop optimizations
  (unrolling, vectorization), instcombine, GVN/SCCP, DCE, target lowering, etc.
  MIR opts are the prep; LLVM is the main course.
- Runs **per CGU in parallel** (see [[10-performance-and-robustness]]).

## 3. LTO (link-time optimization)

`compiler/rustc_codegen_llvm/src/back/lto.rs` — cross-CGU / cross-crate
optimization at link time:
- **Fat LTO**: merge all modules into one and optimize (strongest, slowest).
- **Thin LTO**: use per-module summaries to optimize across modules cheaply.
- Main payoff: **cross-module inlining**.

## Division of labor

- **rustc / MIR opts** = language-aware, lightweight, semantic optimizations +
  IR shaping (prep for the backend).
- **LLVM** = target-aware, heavyweight, low-level optimizations (the `-O2`
  workhorse) + LTO.

## Notes & open questions

- How does CGU partitioning choose boundaries, and how does it interact with
  incremental? (`rustc_monomorphize` partitioning.)
- Worth dumping `rustc -Z mir-opt-level=N --emit=mir` and `--emit=llvm-ir` to
  see both IRs once we can build.

## See also

- [[08-hir-and-mir]] — what MIR is
- [[10-performance-and-robustness]] — CGUs, parallel codegen
- MIR opt guide: https://rustc-dev-guide.rust-lang.org/mir/optimizations.html
- codegen guide: https://rustc-dev-guide.rust-lang.org/backend/codegen.html
