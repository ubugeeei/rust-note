# Monomorphization deep dive

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_monomorphize`
- **Date:** 2026-06-15

## Question

How does rustc turn generic Rust code into concrete, per-type machine-code copies, and how does it group those copies into codegen units (CGUs) that LLVM compiles?

## Short answer

rustc never materializes "monomorphized MIR". Instead it tracks each concrete instantiation as an `Instance` (an `InstanceKind` plus a list of generic `args`), wrapped in a `MonoItem` (`compiler/rustc_middle/src/mono.rs:56`). The *collector* (`compiler/rustc_monomorphize/src/collector.rs`) discovers all needed `MonoItem`s: it finds *roots* by walking the HIR, then walks each item's MIR to find used callees, resolving generic calls against the caller's concrete substitutions until a fixpoint is reached. The *partitioner* (`compiler/rustc_monomorphize/src/partitioning.rs`) then assigns each `MonoItem` to a CGU (one stable + one "volatile" per source module), copies `#[inline]`-able items into every CGU that needs them, merges CGUs down to the requested count, and internalizes symbols. Each concrete instance is codegenned once per CGU it lands in; the actual generic-to-concrete substitution happens lazily during codegen/const-eval, not in this pipeline.

## The core idea: before → after

```rust
// BEFORE (one generic definition)
fn id<T>(x: T) -> T { x }

fn main() {
    id::<i32>(1);
    id::<String>(s);
}
```

Conceptually, codegen produces two concrete copies:

```rust
// AFTER (one MonoItem / Instance per concrete substitution)
fn id__i32(x: i32) -> i32 { x }          // Instance { def: Item(id), args: [i32] }
fn id__String(x: String) -> String { x } // Instance { def: Item(id), args: [String] }
```

Crucially there is no separately-stored MIR for `id__i32`; there is one MIR body for `id`, and an `Instance` that pairs it with `args = [i32]`. The doc on `Instance` spells this out: "Monomorphization happens on-the-fly and no monomorphized MIR is ever created" (`compiler/rustc_middle/src/ty/instance.rs:23`).

## What a `MonoItem` is

A `MonoItem` is anything that becomes a function or global in a CGU's LLVM IR (`compiler/rustc_middle/src/mono.rs:55`):

```rust
pub enum MonoItem<'tcx> {
    Fn(Instance<'tcx>),
    Static(DefId),
    GlobalAsm(ItemId),
}
```

Statics and global asm carry only a `DefId` because they can't be generic; functions carry a full `Instance`. Helpers: `is_generic_fn` checks for non-erasable generic args (`mono.rs:115`), `size_estimate` feeds CGU sizing (`mono.rs:106`), and `instantiation_mode` decides `GloballyShared` vs `LocalCopy` (`mono.rs:132`).

## `Instance` and `InstanceKind`

```rust
pub struct Instance<'tcx> {
    pub def: InstanceKind<'tcx>,        // which kind of body
    pub args: GenericArgsRef<'tcx>,     // the concrete substitutions
}
```

(`compiler/rustc_middle/src/ty/instance.rs:33`). `InstanceKind` (`instance.rs:62`) has many variants. The common one is `Item(DefId)` — an ordinary user `fn`/closure/coroutine body (`:69`). Most others are compiler-generated shims with no user MIR: `Intrinsic` and `Virtual` (the only kinds without callable MIR, `:73`, `:111`), `DropGlue` (`:150`), `CloneShim` (`:159`), `FnPtrShim`, `ClosureOnceShim`, `ReifyShim`, etc.

- `Instance::mono` — trivial instance for a non-generic item; ICEs if it actually has params (`instance.rs:457`).
- `Instance::try_resolve` — resolves `(def_id, args)` to a concrete instance (e.g. `<T as Trait>::method` → the actual impl once `T` is known). Returns `Ok(None)` in a still-polymorphic context (`instance.rs:476`). The collector uses the infallible `expect_resolve`.

## The collector: roots, then recursive reachability

The module header documents the two-phase algorithm: (1) discover roots by walking HIR, (2) from each root, walk MIR to find uses, to a fixpoint (`compiler/rustc_monomorphize/src/collector.rs:53`). Entry: `collect_crate_mono_items` (`:1807`).

- **Strategy** (`partitioning.rs:1131`): `Eager` under `-Clink-dead-code`, otherwise `Lazy` (only reachable instantiations collected).
- **Roots** (`collector.rs:1457`): `RootCollector` over `hir_crate_items`; in `Lazy` mode only non-generic items become roots (`:1656`). Generic fns can never be roots (their args aren't known yet). Roots are filtered to *instantiable* ones (`:1485`).
- **Recursive walk** (`collect_items_rec` `:381`, `collect_items_of_instance` `:1302`): for a `Fn`, fetch the generic MIR with `tcx.instance_mir(...)` and walk it with `MirUsedCollector` (`:1308`). The key substitution step: each type/operand from the MIR goes through `self.monomorphize(...)` = `instantiate_mir_and_normalize_erasing_regions` with the current instance's args (`:692`). So inside `id::<i32>`, a local of type `T` is seen as `i32`. Calls (`visit_terminator` `:815` → `visit_fn_use` `:856`) `expect_resolve` the callee to a concrete `Instance`.
- **Used vs mentioned**: *used* = actually reached; *mentioned* = syntactically present but maybe optimized away (tracked so const-eval errors aren't hidden). Only used items reach output (`:418`).
- **Dedup**: the shared `visited` set ensures each `MonoItem` is recursed at most once across parallel workers (`:572`) — so `id::<i32>` is collected once even if called ten times. Recursion depth bounded via `check_recursion_limit` (`:466`).

## Partitioning into CGUs

`partition` (`partitioning.rs:143`): `place_mono_items` → `merge_codegen_units` → `internalize_symbols` → sort.

- **Instantiation mode** (`mono.rs:132`): forced `GloballyShared` (exactly one external copy) for statics, `global_asm!`, the entrypoint, `#[no_mangle]`/`#[naked]` (`mono.rs:140`). `#[inline(always)]` is always `LocalCopy` (`mono.rs:211`). Roughly: `#[inline]`-able fns → `LocalCopy` (duplicated for inlining); everything else → `GloballyShared`.
- **Placement** (`place_mono_items` `:201`): skip `LocalCopy` items (added via reachability instead). For each `GloballyShared` root, compute a *characteristic `DefId`* (for methods, the self-type's def id, so a type's methods cluster; `:625`), build the **CGU name** from the enclosing module with a `"volatile"` suffix for generic fns under incremental (`:229`, `:689`) — the "two CGUs per module: stable + volatile" scheme. Then pull in transitively reachable `LocalCopy` items as **private internal copies** (`get_reachable_inlined_items` → `inlined: true, Linkage::Internal`, `:262`) — the deliberate duplication that lets LLVM inline within a CGU.
- **Merging / cost tension** (`:317`): bigger CGUs = better LLVM optimization but worse incrementality. `merge_codegen_units` merges until count ≤ `tcx.sess.codegen_units()`, picking the partner with greatest **inlined-item overlap** (`compute_inlined_overlap`) to minimize duplicated code. Non-incremental floor `NON_INCR_MIN_CGU_SIZE = 1800` (`:389`).
- **share-generics dedup**: `should_codegen_locally` returns `false` if an upstream crate already provides the monomorphization (`upstream_monomorphization`), so we link instead of re-codegenning (`collector.rs:1070`).

Debug: `-Zprint-mono-items`, `-Zdump-mono-stats`.

## See also

- [[12-optimization-pipeline]] · [[13-mir-optimizations-catalog]] · [[08-hir-and-mir]] · [[10-performance-and-robustness]] · [[07-glossary]]
