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
