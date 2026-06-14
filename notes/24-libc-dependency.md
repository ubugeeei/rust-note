# What "depends on libc" means

- **Upstream commit:** `4f8ea98e`
- **Area:** `library/std`, `library/core`, the `libc` crate
- **Date:** 2026-06-15

## Question

What does it mean when people say Rust "depends on libc," and how does that relate to `core`, `std`, `no_std`, and choosing a target?

## Short answer

Rust's standard library is layered: `core` is dependency-free and assumes no operating system, `alloc` adds heap types on top of it, and `std` adds everything that needs an OS (files, threads, networking). On most Unix-like targets `std` is implemented by calling into the platform's C standard library: `library/std/Cargo.toml` declares a `libc` crate dependency, and the OS-specific code under `library/std/src/sys` calls functions like `libc::pthread_create`, `libc::read`, and `libc::write`. The `libc` crate is just FFI bindings; the real implementation is whatever C library the target links (glibc for `*-unknown-linux-gnu`, musl for `*-musl`). Targets like `*-unknown-none` have no OS and no libc, which is exactly why `no_std` exists for embedded.

## The layering: core -> alloc -> std

The three crates live side by side: `library/core`, `library/alloc`, `library/std`.

- `core` (`library/core/src/lib.rs`) is "the dependency-free foundation of The Rust Standard Library... the portable glue between the language and its libraries." No allocator, no OS.
- `alloc` sits on top of `core` and adds heap types. `library/alloc/src/lib.rs:71` declares `#![no_std]`; `no_std` crates use it for `Box`, collections, etc.
- `std` depends on both. `library/std/Cargo.toml` lists `alloc = { path = "../alloc" }` and `core = { path = "../core" }`. `std` is the layer that needs an OS.

## std on top of libc: the `sys` layer

The `libc` dependency is per-target in `library/std/Cargo.toml`:

```toml
[target.'cfg(not(all(windows, target_env = "msvc")))'.dependencies]
libc = { version = "0.2.185", default-features = false,
         features = ['rustc-dep-of-std'], public = true }
```

Excluded for Windows MSVC (which uses the Windows API instead), present for Unix-like targets. Platform implementations live under `library/std/src/sys`, split into a per-platform "PAL" (`sys/pal/` with `unix`, `windows`, `wasi`, `unsupported`) plus feature modules (`fs`, `fd`, `net`, `thread`). Real calls into the C library appear directly:

- `library/std/src/sys/thread/unix.rs:98` — `libc::pthread_create(...)`.
- `library/std/src/sys/fd/unix.rs:112` / `:346` — `libc::read` / `libc::write`.
- `library/std/src/sys/fs/unix.rs` — `libc::dirfd`, `libc::fstatat`, `libc::readdir`.

So `std::fs::File::open` or `std::thread::spawn` on Linux ultimately goes through these `libc::*` FFI calls into the system C library, which performs the syscalls.

## The `libc` crate vs glibc/musl

The `libc` crate is *only* the FFI bindings: `extern "C"` declarations and type/constant defs for the system C library's symbols. It does **not** implement `pthread_create`/`read`. The actual implementation is the C standard library the program links, chosen by the target:

- `*-unknown-linux-gnu` → glibc. `compiler/rustc_target/src/spec/targets/x86_64_unknown_linux_gnu.rs` — "64-bit Linux (kernel 3.2+, glibc 2.17+)".
- `*-unknown-linux-musl` → musl. `x86_64_unknown_linux_musl.rs` — "64-bit Linux with musl 1.2.5".
- `*-unknown-none` (bare metal) → no libc. `aarch64_unknown_none.rs` is "Bare ARM64" with `std: Some(false)` (std not available). For contrast, linux-gnu sets `std: Some(true)`.

## Static vs dynamic linking

By default, on glibc Linux the system libc is linked dynamically. The musl target differs: `x86_64_unknown_linux_musl.rs` sets `base.crt_static_default = true`, linking the C runtime statically into a self-contained binary (with a `// FIXME(compiler-team#422)` noting musl should eventually be dynamic by default). This is why `*-musl` is the common choice for portable static Linux binaries, while `*-gnu` depends on the host's glibc at runtime.

## Why this matters

- **Portability:** `std`'s guts are abstracted behind `library/std/src/sys`, so the same source works across OSes; only the `sys` backend changes.
- **Static vs dynamic linking:** controlled at target level (`crt_static_default`) and via `crt-static` — a trade-off between self-containment (musl/static) and shared-library efficiency / system integration (glibc/dynamic).
- **Why `no_std` exists:** on bare-metal targets like `aarch64-unknown-none` there's no OS and no libc, so `std` can't be provided (`std: Some(false)`); such code uses `#![no_std]` on `core` (+ optionally `alloc` with a custom allocator).

## See also

- [[01-repo-layout]] · [[04-bootstrap]] · [[07-glossary]]
