# Reading Notes — Index

Notes are written in English. Each note focuses on one question or topic and
links into the upstream source via `upstream/<path>:<line>` references.

> **Current upstream snapshot:** `4f8ea98e` (rust-lang/rust master, shallow).
> Update this line whenever the submodule is bumped.

## Conventions

- One topic per file. Filename: `NN-topic.md` (zero-padded order).
- Always cite source as `upstream/path/to/file.rs:123` so it is clickable.
- Pin a commit when quoting code: note the upstream commit hash in the file so
  line numbers stay meaningful even after the snapshot moves.
- Use the template in [`_template.md`](./_template.md) for new notes.

## Topics

| # | Topic | Status |
|---|-------|--------|
| 01 | [Repo layout: `src/` vs `compiler/`](./01-repo-layout.md) (incl. why `x`/`x.py`) | done |
| 02 | [Codebase scale & test counts](./02-scale-and-tests.md) | done |
| 03 | [Who contributes & how Rust is funded](./03-governance-and-business.md) | done |
| 04 | [How bootstrap works (stage0→1→2)](./04-bootstrap.md) | done |
| 05 | [How to do a full build](./05-building.md) | done |
| 06 | [How to read the compiler](./06-reading-the-compiler.md) | done |
| 07 | [Glossary](./07-glossary.md) | living |
| 08 | [HIR & MIR: roles and concrete examples](./08-hir-and-mir.md) | done |
| 09 | [From source code to AST (lexing & parsing)](./09-source-to-ast.md) | done |
| 10 | [Performance & robustness engineering](./10-performance-and-robustness.md) | done |
| 11 | [How editions are implemented](./11-editions.md) | done |
| 12 | [Optimization: MIR passes & where LLVM is invoked](./12-optimization-pipeline.md) | done |
| 13 | [MIR optimizations & transforms — a visual catalog](./13-mir-optimizations-catalog.md) | done |
| 14 | [CFG simplification (`SimplifyCfg`) deep dive](./14-cfg-simplification.md) | done |
| 15 | [Branch optimizations deep dive](./15-branch-optimizations.md) | done |
| 16 | [Type inference: how it's implemented](./16-type-inference.md) | done |
| 17 | [Lifetimes: region inference & NLL](./17-lifetimes.md) | done |
| 18 | [Monomorphization deep dive](./18-monomorphization.md) | done |
| 19 | [The HIR data format](./19-hir-format.md) | done |
| 20 | [The MIR data format](./20-mir-format.md) | done |
| 21 | [Arena allocation](./21-arena.md) | done |
| 22 | [AST lowering (AST -> HIR)](./22-ast-lowering.md) | done |
| 23 | [Rust's relationship with GCC](./23-gcc-relationship.md) | done |
| 24 | [What "depends on libc" means](./24-libc-dependency.md) | done |
| 25 | [SSA (Static Single Assignment)](./25-ssa.md) | done |
| 26 | [Incremental compilation (dep graph)](./26-incremental.md) | done |
| 27 | [What "passes" are in rustc](./27-passes.md) | done |
| 28 | [Pattern analysis (exhaustiveness & usefulness)](./28-pattern-analysis.md) | done |
| 29 | [`rustc_type_ir`: type-system IR abstraction](./29-type-ir.md) | done |
| 30 | [The state of Rust's ABI](./30-abi-status.md) | done |
| 31 | [SIMD in Rust](./31-simd.md) | done |
| 32 | [Static vs dynamic dispatch](./32-static-dynamic-dispatch.md) | done |
| 33 | [Governance, leadership & participation](./33-governance-participation.md) | done |
| 34 | [Who manages Rust releases](./34-release-management.md) | done |
| 35 | [Why Rust builds are slow](./35-why-builds-are-slow.md) | done |
| 36 | [Notable Rust security advisories (CVEs)](./36-cve-summary.md) | done |
| 37 | [rustc CI: jobs, runners, timing](./37-ci.md) | done |
| 38 | [How Rust implements multithreading](./38-multithreading.md) | done |
| 39 | [How closures are implemented](./39-closures.md) | done |
| 40 | [How smart pointers are implemented](./40-smart-pointers.md) | done |
| 41 | [Module visibility & item privacy](./41-visibility.md) | done |

## Suggested reading path (rustc)

A rough map of the compilation pipeline, to orient future notes:

1. **Driver & CLI** — `compiler/rustc_driver`, how a compile is kicked off
2. **Lexing & Parsing** — `compiler/rustc_lexer`, `compiler/rustc_parse` → AST
3. **AST lowering** — AST → HIR (`compiler/rustc_ast_lowering`)
4. **Name resolution** — `compiler/rustc_resolve`
5. **Type checking & inference** — `compiler/rustc_hir_typeck`, `compiler/rustc_infer`
6. **MIR** — building & optimizing (`compiler/rustc_mir_build`, `rustc_mir_transform`)
7. **Borrow checking** — `compiler/rustc_borrowck`
8. **Codegen** — `compiler/rustc_codegen_llvm`, `rustc_codegen_ssa`

(We will adjust this as we actually explore the tree.)
