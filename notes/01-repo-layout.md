# Repository layout: `src/` vs `compiler/`

- **Upstream commit:** `4f8ea98e`
- **Area:** top-level tree
- **Date:** 2026-06-14

## Question

What is the difference between `src/` and `compiler/`? What even *is* `src/`?
Naively you'd expect the compiler's source to live in `src/`, but it doesn't.

## Short answer

`compiler/` holds the **rustc compiler itself** (the `rustc_*` crates).
`src/` holds **everything around it** — the build system, vendored tools,
docs, CI, and the LLVM submodule. `src/` is *not* the compiler. Historically
everything lived under `src/`; the compiler was later moved out to `compiler/`
and the standard library to `library/`, leaving `src/` as the "rest".

## Walkthrough

Top-level directories:

| Dir          | Contents                                  | In short              |
|--------------|-------------------------------------------|-----------------------|
| `compiler/`  | 75 `rustc_*` crates                       | the rustc compiler    |
| `library/`   | `core`, `alloc`, `std`, ...               | the standard library  |
| `src/`       | bootstrap, tools, doc, ci, llvm-project   | everything else       |
| `tests/`     | the test suite                            | tests                 |
| `LICENSES/`  | license texts                             | —                     |

`upstream/src/README.md` says it plainly:

> This directory contains some source code for the Rust project, including:
> - The bootstrapping build system
> - Various submodules for tools, like cargo, tidy, etc.

### What's inside `src/`

- `src/bootstrap/` — Rust's own build system, the implementation behind `./x.py`.
  Because rustc is itself written in Rust, it must be built in *stages*
  (a prebuilt "stage0" compiler builds stage1, which builds stage2...).
  That staging logic lives here.
- `src/tools/` — peripheral tools, many of them git submodules of their own
  repos: `cargo`, `clippy`, `miri`, `rustfmt`, `rust-analyzer`, `rustdoc`,
  `tidy` (the lint/style checker), `compiletest` (the test runner), etc.
- `src/librustdoc/` — the actual implementation of rustdoc.
- `src/llvm-project/` — the LLVM backend, a large submodule.
- `src/ci/`, `src/doc/`, `src/etc/`, `src/stage0/`, `src/version` — CI config,
  official docs, misc config, the stage0 bootstrap compiler info, version.

### Why it's split this way (history)

Originally **everything** lived under `src/`: the compiler was `src/librustc_*`
and the standard library was `src/libstd` etc. The tree grew huge, so a large
reorganization moved:

- the compiler crates out to `compiler/` (renamed into the `rustc_*` layout), and
- the standard library out to `library/`.

What was left behind in `src/` — build infrastructure, tooling, and docs — is
what you see today. So when reading compiler internals, look in `compiler/`,
**not** `src/`.

## Aside: why is the build tool called `x` / `x.py`?

`x.py` is the **entry point to bootstrap** (the build system). It's a thin
wrapper — its own header says *"This file is only a 'symlink' to bootstrap.py,
all logic should go there."* There are three variants: `x.py` (cross-platform),
`x` (Unix), `x.ps1` (Windows). The `src/tools/x` crate is a tiny helper so you
can run `x` from any subdirectory; its README only says *"`x` invokes `x.py`"*.

There is **no officially documented reason** for the letter `x` (neither the
repo nor the rustc-dev-guide records an etymology). The commonly understood
rationale is purely ergonomic: a single, easy-to-type character that acts as a
generic placeholder, so `x build` / `x test` / `x check` all read naturally.

## Notes & open questions

- Mental model: `compiler/` = the program we're studying; `src/` = the scaffolding
  that builds and ships it; `library/` = the code that program compiles for users.
- Follow-up to read next: how `src/bootstrap` stages a build (stage0/1/2), and
  the entry point in `compiler/rustc_driver`.

## See also

- rustc dev guide: https://rustc-dev-guide.rust-lang.org/about-this-guide.html
- [[_template]]
