# Why Rust builds are slow

- **Upstream commit:** `4f8ea98e`
- **Area:** cross-cutting (monomorphization, LLVM, linking)
- **Date:** 2026-06-15

## Question

Where does the time actually go when `rustc` compiles a crate, and which settings/tools move the needle?

## Short answer

Most wall-clock cost is not in rustc's front end (parse, infer, borrow check) but downstream: rustc *monomorphizes* every generic into concrete copies, so a little generic source expands into a lot of LLVM IR, and LLVM spends the bulk of the time optimizing/codegenning it. The crate is the unit of compilation (dependencies serialize), and within a crate codegen-unit (CGU) splitting trades optimization for parallelism. Mitigations attack one lever: do less (incremental, `-Zshare-generics`, lower `opt-level`), parallelize (parallel front end, more CGUs, sccache), or use a cheaper backend/linker (Cranelift, lld/mold).

## Cost drivers

### 1. Monomorphization — generics become many concrete copies
A generic fn is compiled once *per distinct set of type arguments*. The collector doc: "Every non-generic, non-const function maps to one LLVM artifact. Every generic function can produce from zero to N artifacts" (`rustc_monomorphize/src/collector.rs`). This multiplier turns compact source into a large pile of IR. It even pulls in instantiations of generics from *other* crates, so you pay codegen cost for `std`/dependency generics you merely use. See [[18-monomorphization]].

### 2. LLVM backend time often dominates
Each mono-item is handed to LLVM as IR; because there's so much (#1) and LLVM runs a full optimization pipeline ([[12-optimization-pipeline]]), the LLVM half is frequently the largest single bucket. Key constraint: "LLVM optimization does not work across module boundaries" (`partitioning.rs`) — which is why CGU splitting is a trade-off, not free parallelism.

### 3. The crate is the unit; CGUs are the boundary
A crate compiles as a whole, so dependent crates form a serial chain (cargo parallelizes *across* independent crates, not the dependency spine). Inside a crate, mono-items are partitioned into CGUs. `partitioning.rs`: more CGUs = better incremental reuse + parallelism, but "since LLVM optimization does not work across module boundaries... highly granular partitioning would lead to very slow runtime code." The heuristic is ~two CGUs per module (stable + volatile). See [[26-incremental]].

### 4. Trait resolution / type inference
Trait selection, obligations, coherence run in the front end. Usually a smaller slice than LLVM, but trait-heavy/deeply-generic code can be costly. Its *output* — the resolved instances — feeds the #1 explosion.

### 5. Linking time (many CGUs / debuginfo)
Per-CGU object files must be linked; more CGUs = more objects, and debuginfo inflates object size and linker work. rustc invokes the platform linker by default, often slow. Pure serial tail latency.

### 6. Proc-macros and build scripts
Proc-macros are crates that must be compiled *then executed* during your build; `build.rs` likewise. Both add serial, non-cacheable work and can pull heavy deps (e.g. `syn`).

### 7. Heavy abstraction → lots of pre-optimization IR
Iterator chains, async state machines, closures, smart-pointer wrappers are "zero-cost" only *after* optimization; *before* it each layer emits its own IR, which LLVM must inline/fold away — landing on #2.

**Mental model:** *Rust does a lot of monomorphized work and hands huge IR to LLVM.* The front end decides *what* to instantiate; monomorphization multiplies it; LLVM pays for the volume; the linker pays at the end.

## Mitigations

| Lever | Mechanism | Attacks |
|---|---|---|
| Incremental compilation | Reuse unchanged CGUs | #2, #3 |
| Parallel front end | `rustc_thread_pool` (`rustc_interface/src/util.rs`) | #4 |
| Cranelift backend | Cheaper codegen for debug builds (`rustc_codegen_cranelift`) | #2 |
| `-Zshare-generics` | Don't re-emit upstream monomorphizations (`upstream_monomorphizations`) | #1 |
| Faster linkers (lld/mold) | Replace slow system linker (`config.rs:427`) | #5 |
| `-C codegen-units=N` | Tune parallelism vs optimization | #2, #3 |
| `opt-level` / `debug` | Less optimization / less debuginfo | #2, #5 |
| sccache | Cache artifacts across builds | #1–#5 |

`-Zshare-generics`'s default tracks opt-level: on for `No/Less/Size/SizeMin` (debug — prioritize compile time), off for `More/Aggressive` (let LLVM specialize) — `share_generics` at `rustc_session/src/config.rs:1489`.

## See also

- [[18-monomorphization]] · [[12-optimization-pipeline]] · [[26-incremental]] · [[10-performance-and-robustness]] · [[07-glossary]]
