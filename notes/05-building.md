# How to do a full build

- **Upstream commit:** `4f8ea98e`
- **Area:** `x.py`, `INSTALL.md`, `bootstrap.toml`
- **Date:** 2026-06-14

## Question

How do I do a full build of rustc from source?

## Short answer

Two steps: `./x.py setup` (generates `bootstrap.toml`, pick a profile) then
`./x.py build`. `build` defaults to **stage1**; use `--stage 2` for a full
self-hosted compiler. For day-to-day work you usually scope it down with
`./x.py check` or `./x.py build compiler/rustc`.

## Walkthrough

### Dependencies (from `INSTALL.md`)

- `python` 3, `git`, a **C compiler** (`cc` is enough for host builds),
  `curl` (not on Windows).
- Building LLVM from source additionally needs `g++`/`clang++`, `cmake`,
  `ninja`, etc. — but normally bootstrap **downloads a CI-built LLVM**, so you
  don't need these.

### Steps

```sh
# 1. configure interactively (profile, LSP, git hook) -> writes bootstrap.toml
./x.py setup

# 2. full build (defaults to stage1)
./x.py build

# 3. (optional) install rustc/rustdoc/cargo into $PREFIX/bin
./x.py install
```

- `./x.py setup` writes the config file **`bootstrap.toml`**. Pick a profile:
  `compiler` (hacking on rustc), `library` (hacking on std), `tools`, `dist`.
- `./x.py build` builds up to **stage1** by default. For the fully self-hosted
  compiler use `./x.py build --stage 2`.
- Instead of interactive setup you can script config via
  `./configure --set <key>=<val> ...` (INSTALL.md shows a cross-build example).
  `./configure` also wires up a Makefile that just forwards to `x.py`.

### Scoped builds (what you actually use while developing)

```sh
./x.py check                  # type-check only — fast, most common
./x.py build compiler/rustc   # just the compiler
./x.py build library          # just std
./x.py test tests/ui          # run the UI test suite
```

## Notes & open questions

- A full build (incl. LLVM) can take tens of minutes to hours and needs tens of
  GB of disk. Don't run it casually.
- Our `upstream` is a **shallow** submodule; if we ever actually build, a full
  clone is safer.

## See also

- [[04-bootstrap]] — what stage0/1/2 mean
- [[01-repo-layout]] — `x.py` lives at the root, logic in `src/bootstrap`
- how-to-build-and-run: https://rustc-dev-guide.rust-lang.org/building/how-to-build-and-run.html
