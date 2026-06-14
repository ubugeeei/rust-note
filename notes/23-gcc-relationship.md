# Rust's relationship with GCC

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_codegen_gcc`, `compiler/rustc_codegen_ssa`, `src/gcc`
- **Date:** 2026-06-15

## Question

What does "Rust and GCC" actually refer to inside the `rust-lang/rust` tree, and how do the GCC codegen backend, the bundled GCC source, and the separate gccrs project differ?

## Short answer

There are three distinct things that are easy to conflate. `compiler/rustc_codegen_gcc` is an *alternative rustc codegen backend* that lowers rustc's MIR-derived IR through libgccjit/GCC instead of LLVM, while still using the normal rustc frontend. `src/gcc` is a git submodule pointing at `rust-lang/gcc`, i.e. the GCC source itself, used to build the GCC libraries that backend needs. gccrs is a completely separate effort — a Rust front-end living *inside* GCC (not in this repo). All rustc backends, including the default LLVM one, implement a shared interface defined in `compiler/rustc_codegen_ssa`, which is what makes backends pluggable.

## 1. Pluggable codegen backends (the `rustc_codegen_ssa` interface)

rustc separates its frontend (parse, typeck, borrowck, MIR) from final code generation. The shared backend contract lives in `compiler/rustc_codegen_ssa`, whose central trait is:

- `compiler/rustc_codegen_ssa/src/traits/backend.rs:36` — `pub trait CodegenBackend { fn name(&self) -> &'static str; ... }`

`rustc_codegen_ssa` provides the arch-agnostic machinery (mono collection, MIR-to-backend-IR scaffolding, the `CodegenBackend`/builder/type traits); each concrete backend implements it. Confirmed implementors:

- `compiler/rustc_codegen_llvm/src/lib.rs:217` — `LlvmCodegenBackend` (the **default**, via LLVM)
- `compiler/rustc_codegen_cranelift/src/lib.rs:123` — `CraneliftCodegenBackend` (pure-Rust backend for fast debug builds)
- `compiler/rustc_codegen_gcc/src/lib.rs:193` — `GccCodegenBackend` (via libgccjit/GCC)

Backends are loaded as dynamic libraries at runtime: rustc looks up a `__rustc_codegen_backend` symbol in the chosen dylib (`compiler/rustc_interface/src/util.rs:313`), found in a `codegen-backends` folder of the sysroot (`util.rs:517`). Which backends get built is selected in bootstrap: `bootstrap.example.toml:806` (`#rust.codegen-backends = ["llvm"]`); `enum CodegenBackendKind { Llvm, Cranelift, Gcc, ... }` (`src/bootstrap/src/lib.rs:127`).

## 2. `compiler/rustc_codegen_gcc` (the GCC *backend*)

A rustc backend that translates code through **libgccjit** (GCC's embeddable library — despite "jit", used here for AOT too), reusing GCC's optimizer and arch-specific codegen. Same rustc frontend; only the final lowering changes.

- Directory exists with `Cargo.toml`, `build_system/`, `libgccjit.version`, `example/`, etc.
- `Cargo.toml` depends on the `gccjit` crate (Rust bindings to libgccjit): `gccjit = { version = "3.3.0", features = ["dlopen"] }`.
- `libgccjit.version` pins the expected libgccjit version.

## 3. `src/gcc` (the GCC *source*, a submodule)

To build the GCC libraries the backend links against, the tree vendors GCC as a submodule (`.gitmodules`):

```
[submodule "src/gcc"]
    path = src/gcc
    url = https://github.com/rust-lang/gcc.git
    shallow = true
```

`git submodule status src/gcc` reports the pinned SHA with a leading `-` (not checked out in this clone — only the pinned commit is recorded). URL is `rust-lang/gcc` (a rust-lang fork/mirror of GCC). So `src/gcc` provides the GCC source; `rustc_codegen_gcc` is the rustc-side glue driving it via libgccjit.

## 4. gccrs (disambiguation only — NOT in this repo)

gccrs is a separate project: a **Rust front-end built into GCC** (`gcc` itself learning to compile Rust), developed in the GCC tree (gcc.gnu.org), not in `rust-lang/rust`. It's the mirror image of the backend above:

- `rustc_codegen_gcc` = rustc frontend + GCC backend.
- gccrs = GCC frontend-for-Rust + GCC backend (does not use rustc at all).

No gccrs sources are in this checkout (expected). Beyond this distinction, no detailed gccrs claims are made here.

## Why an alternative GCC backend matters

> Standard project rationale; the source confirms the *mechanism* (pluggable backends, the GCC backend crate, the GCC submodule) but does not argue these benefits — treat as context.

- **Architecture/platform coverage:** GCC supports targets LLVM does not (or supports less maturely), extending where Rust code can compile.
- **GPL / distro ecosystem alignment:** eases integration with GNU-toolchain build systems and distributions.
- **Toolchain diversity:** a second independent codegen path (plus Cranelift as a third) reduces single-vendor reliance on LLVM — valuable for trusting-trust/reproducibility and resilience.

## See also

- [[12-optimization-pipeline]] · [[01-repo-layout]] · [[04-bootstrap]] · [[07-glossary]]
