# Performance & robustness engineering in rustc

- **Upstream commit:** `4f8ea98e`
- **Area:** cross-cutting (compiler + project infra)
- **Date:** 2026-06-14

## Question

What tricks does rustc/the Rust project use for performance and for
availability/robustness?

## Short answer

Performance leans on a **demand-driven query system with on-disk incremental
caching**, heavy **interning** (`Symbol`, `Ty`), **arena** allocation,
purpose-built data structures (`FxHashMap`, `IndexVec`), a **parallel
front-end**, and **codegen-unit** partitioning. Robustness/availability comes
from a **never-merge-red bors queue**, **Crater** ecosystem testing, continuous
**perf** tracking, **feature gates + release trains + editions** for
compatibility, a huge **UI test** suite, and graceful **ICE handling** + parser
error recovery.

## A. Performance

1. **Query system + incremental compilation** — `rustc_query_system`,
   `rustc_query_impl`, `rustc_incremental`. Everything is a query (computed
   lazily, cached, persisted to disk); a red/green dependency graph skips
   recomputation of unchanged work.
2. **Interning** — identifiers become `Symbol` (a small `SymbolIndex`),
   `compiler/rustc_span/src/symbol.rs:2617`, interner at `:2766`. Types/consts
   are interned in `TyCtxt`, so equality is pointer/id comparison, not deep.
3. **Arena allocation** — `rustc_arena`. Bump-allocate the many HIR/MIR nodes
   with a shared lifetime; free them all at once. Avoids per-node malloc/free.
4. **Purpose-built data structures** — `rustc_data_structures`, `rustc_index`:
   - `FxHashMap`: fast non-cryptographic hash (no trust boundary in a compiler).
   - `IndexVec` (`rustc_index/src/vec.rs:40`): typed integer indices instead of
     pointers — cache-friendly, compact.
   - bitsets, `SmallVec` (stack storage when small), etc.
5. **Parallel front-end** — `rustc_thread_pool` (Rayon-derived) parallelizes
   front-end work like type checking.
6. **Codegen units (CGUs)** — a crate is split into multiple units so LLVM
   codegen runs in parallel and is incremental.
7. **Prebuilt CI LLVM** — download a CI-built LLVM instead of compiling it (see
   [[05-building]]).
8. **jemalloc** — the compiler links jemalloc to cut allocation overhead
   (`rust.jemalloc`).
9. **Lazy crate metadata** — `rustc_metadata` decodes dependency `.rmeta` info
   on demand, not all up front.
10. **Built-in profiling** — `rustc_data_structures/src/profiling.rs`,
    `-Z self-profile` to see where time goes.
11. **Compact spans** — `rustc_span` keeps `Span` small, spilling large span
    data into a side table to save memory.

## B. Robustness / availability / quality

1. **bors merge queue ("not rocket science rule")** — nothing merges to `main`
   unless the full CI passes; `main` always builds.
2. **Crater** — rebuilds a change against the entire crates.io ecosystem to
   catch regressions in real-world crates before release.
3. **Continuous perf tracking** — `src/tools/rustc-perf` → perf.rust-lang.org;
   performance regressions are flagged per-PR. Speed is a gated property too.
4. **Release trains + feature gates** — `rustc_feature`; nightly→beta→stable on
   a 6-week cycle. `#![feature(...)]` ensures unstable features can't reach
   stable; `#[stable]`/`#[unstable]` track stability.
5. **Editions** (2015/2018/2021/2024) — evolve the language without breaking
   backwards compatibility.
6. **Huge UI test suite + known-crash tracking** — `tests/ui` (~20k) diffs
   compiler output; `tests/crashes` tracks known ICEs so fixes are detected.
7. **ICE (internal compiler error) handling** — `rustc_driver_impl/src/lib.rs`:
   panics are caught via `catch_fatal_errors` (`:1377`), an ICE report is
   written (`ICE_PATH` `:1384`), and the user is pointed at a prefilled bug
   report URL (`:117`). A crash still gives the user a path forward.
8. **Parser error recovery** — `rustc_parse/src/parser/diagnostics.rs`: don't
   stop at the first error; build a partial AST and keep reporting, so many
   diagnostics surface at once.
9. **Determinism under parallelism** — `rustc_data_structures/src/unord.rs`'s
   `UnordMap`/`UnordSet` remove hashmap iteration-order nondeterminism, keeping
   output reproducible even with parallel/incremental builds.
10. **Bootstrap reproducibility** — compare stage2 vs stage3 to verify the
    self-hosted compiler is consistent (see [[04-bootstrap]]).
11. **Internal quality gates** — `rustc_sanitizers`, `src/tools/tidy` (lints the
    project's own source/style), deny-warnings.

## Notes & open questions

- Deep-dive candidates: the query system internals (`rustc_query_system`), and
  how the incremental dep-graph's red/green marking actually works.
- How CGU partitioning decides boundaries (`rustc_monomorphize`).

## See also

- [[06-reading-the-compiler]] — the query system as a core mental model
- [[07-glossary]] · [[04-bootstrap]] · [[05-building]]
