# Codebase scale & test counts

- **Upstream commit:** `4f8ea98e`
- **Area:** whole tree (measured with `tokei` + `find`)
- **Date:** 2026-06-14

## Question

How big is the codebase, and how many tests are there?

## Short answer

The compiler itself (`compiler/`) is ~650k lines of Rust; counting the standard
library and tooling, the repo holds ~2.4M lines of Rust (~3.3M including all
languages). The test suite has ~25,800 `.rs` test files (~42,400 files total
incl. expected-output files), dominated by ~20,600 UI tests in `tests/ui`.

## Walkthrough

### Code size (Rust code lines, excluding comments/blanks)

| Target                         | Rust code | Files  | Note                         |
|--------------------------------|-----------|--------|------------------------------|
| `compiler/` (rustc itself)     | ~648,651  | ~2,100 | +~75k comment lines          |
| `library/` (std library)       | ~1.29M    | ~2,200 | includes its own tests       |
| whole repo, Rust only          | ~2.38M    | ~11,500| compiler + library + src     |
| whole repo, all languages      | ~3.33M    | ~14,000| C++/Python/Markdown/etc.     |

Mental anchor: **~650k lines for the compiler alone**, ~2.4M lines of Rust total.

### Tests (`tests/`)

| Category                | Count   | What it is                                   |
|-------------------------|---------|----------------------------------------------|
| all files in `tests/`   | ~42,400 | includes `.stderr`/`.stdout` expected output |
| `.rs` test files        | ~25,803 | best proxy for "number of tests"             |
| └ `tests/ui`            | ~20,591 | compile & diff against expected diagnostics  |
| └ `tests/mir-opt`       | 1,679   | MIR optimization checks                       |
| └ `tests/run-make`      | 1,496   | Makefile-driven integration tests            |
| └ `tests/coverage`      | 309     | coverage instrumentation                      |
| └ `tests/incremental`   | 214     | incremental compilation                       |
| └ `tests/debuginfo`     | 185     | debug info / debugger integration             |
| └ `tests/pretty`        | 150     | pretty-printing round-trips                   |

Note: directory names changed over time — codegen tests live in
`tests/codegen-llvm`, assembly in `tests/assembly-llvm`, and rustdoc tests are
split across `tests/rustdoc-html`, `tests/rustdoc-ui`, `tests/rustdoc-js`, etc.

A single `.rs` UI test can declare multiple `revisions`, so the real number of
*assertions* is higher than the file count; file count is the practical proxy.

## Notes & open questions

- The UI test suite is the heart of rustc's testing: each test is a small
  program whose compiler output (errors/warnings) is diffed against a checked-in
  `.stderr`. Worth a dedicated note when we study `compiletest`.

## See also

- [[01-repo-layout]] — where `tests/` and `compiler/` sit
