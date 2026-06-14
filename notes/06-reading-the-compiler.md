# How to read the compiler

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/` (75 `rustc_*` crates)
- **Date:** 2026-06-14

## Question

How should we actually read `compiler/`? Where do we start in 75 crates?

## Short answer

Don't read top-to-bottom. First grasp **two structures**: the **pipeline**
(source → tokens → AST → HIR → typeck → MIR → codegen, which crate names mirror)
and the **query system** (rustc is demand-driven, centered on `TyCtxt` in
`rustc_middle`). Then start at the entry point `compiler/rustc/src/main.rs` →
`rustc_driver_impl` → `rustc_interface/src/passes.rs` (the "table of contents"
of phases), and walk the pipeline crates front-to-back.

## Walkthrough

### Two axes to understand first

1. **The pipeline (vertical flow).** Crate names roughly follow compilation
   order. Reading them in order = reading the compiler in order.
2. **The query system (horizontal mechanism).** rustc does **not** run strictly
   top-down; it's **demand-driven**: you ask for a result (a "query"), it's
   computed lazily and cached. The hub is `TyCtxt` in `rustc_middle`. Without
   this mental model the call graph looks confusing.

### Suggested reading order

**Step 0 — see the entry point (short):**
- `compiler/rustc/src/main.rs` — almost empty; just calls into the driver.
- `compiler/rustc_driver_impl/src/lib.rs` — the real CLI driver.
- `compiler/rustc_interface/src/passes.rs` — **drives each phase in order**;
  this is the best single file for the big picture.

**Step 1 — foundations (data structures):**
- `rustc_span` (source locations, interned `Symbol`s)
- `rustc_data_structures` (arenas, interning, maps)
- `rustc_middle::ty` (`TyCtxt`, type representations)

**Step 2 — the pipeline, front to back:**
`rustc_lexer` → `rustc_parse` (→ `rustc_ast`) → `rustc_ast_lowering`
(→ `rustc_hir`) → `rustc_resolve` (name resolution) →
`rustc_hir_typeck` / `rustc_infer` (type inference) →
`rustc_mir_build` / `rustc_mir_transform` (MIR) →
`rustc_borrowck` (borrow checking) →
`rustc_codegen_ssa` / `rustc_codegen_llvm` (codegen).

### Tips for reading

- **Trace one tiny input**, e.g. compiling `fn main() {}`, starting from
  `passes.rs`.
- **Iterate fast with `./x.py check`** (no full build). Get `rust-analyzer`
  working for go-to-definition — it's a huge multiplier in 75 crates.
- **See the real IRs** with dump flags: `rustc -Z unpretty=hir`,
  `-Z unpretty=mir`, and `RUSTC_LOG=...` for tracing.
- **Use the rustc-dev-guide as the map** — each phase has a chapter; read it
  before the code to avoid getting lost.
- **Dependency direction:** lower layers (`rustc_span`, `rustc_middle`) are
  shared by upper-layer passes; `rustc_middle` is the common hub.

## Notes & open questions

- Next concrete session idea: open `rustc_interface/src/passes.rs` together and
  map each call to its crate, building a real per-phase index.
- Then a dedicated note on the **query system** (`rustc_middle`,
  `rustc_query_impl`) — it's the concept everything else hangs off.

## See also

- [[01-repo-layout]] — compiler vs src vs library
- [[04-bootstrap]] — building the compiler we're reading
- rustc-dev-guide overview: https://rustc-dev-guide.rust-lang.org/overview.html
