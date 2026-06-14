# The state of Rust's ABI

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_abi`, `compiler/rustc_target`
- **Date:** 2026-06-15

## Question

What does "the Rust ABI is unstable" actually mean, and how does rustc model the two things that phrase conflates — type *layout* and *calling conventions*?

## Short answer

"ABI" covers both *type layout* (how a value is arranged in memory) and *calling conventions* (how a function passes args/returns). The default `extern "Rust"` / `repr(Rust)` is deliberately **unspecified**: the compiler may reorder struct fields and pick any calling convention, and may change those between versions — which is why two Rust artifacts must be built with the *same* compiler to interoperate, and why `#[repr(C)]` exists as the stable interop story. `rustc_abi` computes layout (`LayoutData`, `Variants`/`TagEncoding`, niche optimization). Calling conventions live in `rustc_target::callconv` (`FnAbi`/`ArgAbi`/`PassMode`) on top of the `ExternAbi`/`CanonAbi` enums in `rustc_abi`. There is **no** stable cross-version Rust ABI today; no `crabi`/`extern "crabi"` was found in this tree.

## What rustc means by "ABI"

The `rustc_abi` crate doc spells out the ambiguity (`lib.rs:7`): ABI technically covers object format, in-memory layout of types, and procedure calling conventions. It notes "the Rust ABI is unstable" alludes to *either or both* of (`lib.rs:21`): `repr(Rust)` types have mostly-unspecified layout, and `extern "Rust" fn` has an unspecified calling convention. Two distinct axes.

## Axis 1 — Layout ABI (`rustc_abi`)

**`repr` flags and reordering.** `ReprFlags` (`lib.rs:81`) carries `IS_C`, `IS_SIMD`, `IS_TRANSPARENT`, `IS_LINEAR`, `RANDOMIZE_LAYOUT`. The gate is `ReprOptions::inhibit_struct_field_reordering` (`lib.rs:217`):

```rust
pub fn inhibit_struct_field_reordering(&self) -> bool {
    self.flags.intersects(ReprFlags::FIELD_ORDER_UNOPTIMIZABLE) || self.int.is_some()
}
```

`FIELD_ORDER_UNOPTIMIZABLE = IS_C | IS_SIMD | IS_SCALABLE | IS_LINEAR` (`lib.rs:98`). A plain `repr(Rust)` struct is **not** in that set, so its fields may be reordered; `repr(C)` pins the order.

**The reordering** happens in `LayoutCalculator` (`layout.rs:1111`): `let optimize_field_order = !repr.inhibit_struct_field_reordering();`. When enabled, fields are sorted largest-alignment-first, so a `repr(Rust)` struct's memory order can differ from declaration order (it keeps an `in_memory_order` map, `layout.rs:1110`). Under `-Z randomize-layout`, `repr(Rust)` field order is *shuffled* with a deterministic seed (`layout.rs:1124`) — a tool to catch code assuming an unpromised layout.

**Niche optimization / `Option<&T>`.** Layouts are `LayoutData` (`lib.rs:2102`) with `Variants` (`lib.rs:1937`) and `TagEncoding` (`lib.rs:1964`): a multi-variant enum is either `Direct` (a real tag) or `Niche` (discriminant hidden in an invalid bit-pattern). The doc's canonical case (`lib.rs:1984`): `Option<&T>` needs no tag because `&T` can never be null — `None` is the null pointer, `Some` the identity — so `Option<&T>` is pointer-sized (the null-pointer optimization). `inhibit_enum_layout_opt` (`lib.rs:209`) turns this off for `repr(C)` enums.

## Axis 2 — Calling-convention ABI (`rustc_target::callconv`)

**Surface enums in `rustc_abi`.** `ExternAbi` (`extern_abi.rs:20`): `Rust`, `RustCall`, `C { unwind }`, `System`, `SysV64`, `Win64`, `Aapcs`, `Stdcall`, `Custom`, `Unadjusted`, etc. Several doc comments flag instability (`Unadjusted` "directly uses Rust types... can become ABI-incompatible", `:57`; `RustTail`/`RustPreserveNone` "not stable", `:46`). `CanonAbi` (`canon_abi.rs:22`) is the canonicalized directive for codegen, collapsing syntactic distinctions; `is_rustic_abi` (`:60`) groups the Rust-family conventions.

**`PassMode`** (`rustc_target/src/callconv/mod.rs:40`): `Ignore` (ZST), `Direct` (register), `Pair` (ScalarPair in two registers), `Cast` (coerced), `Indirect` (hidden pointer, `byval`/by-ref). Default comes from `BackendRepr` in `ArgAbi::new` (`:392`). `FnAbi` bundles `args`, `ret`, `c_variadic`, `conv: CanonAbi`, `can_unwind` (`:604`).

**Where `FnAbi` is built.** `fn_abi_new_uncached` in `rustc_ty_utils/src/abi.rs:561`, then `adjust_for_rust_abi` (`mod.rs:728`) — the heart of the *unstable* Rust convention, dispatching to per-arch `compute_rust_abi_info` and applying Rust-only rules (e.g. returning >2-register values via a return pointer). `adjust_for_foreign_abi` (`mod.rs:641`) routes to the per-target files (`x86_64.rs`, `aarch64.rs`, …) implementing the *specified*, stable platform conventions.

## Axis 3 — Stability status

- **No stable cross-version Rust ABI.** Field reordering + internal arch-specific Rust call adjustments mean `repr(Rust)`/`extern "Rust"` has no cross-version guarantee — separately-compiled Rust artifacts must use the same compiler version (same reason Rust has no stable dylib-plugin/proc-macro ABI).
- **`#[repr(C)]` is the stable story.** `IS_C` inhibits both field reordering (`lib.rs:219`) and enum niche opt (`lib.rs:209`), giving fixed C-compatible layout; with `extern "C"` it yields a fully specified FFI ABI.
- **crABI:** **Not present in this tree.** `grep -rn 'crabi\|Crabi' compiler/` returned nothing at this commit. The RFC-stage "crABI" (a candidate stable, richer-than-C interop ABI) is unverifiable here — flagged as uncertain. The closest in-tree relative is `ExternAbi::Unadjusted`, an unstable impl detail.

## Canonical examples (verified mechanics)

- `repr(Rust)` `struct S { a: u8, b: u64, c: u8 }` may put the `u64` first (alignment-descending sort, `layout.rs:1201`) — memory order ≠ declaration order; `#[repr(C)]` forbids this.
- `Option<&T>` is pointer-sized via `&T`'s non-null niche (`TagEncoding::Niche`, `lib.rs:1984`).

## See also

- [[24-libc-dependency]] · [[12-optimization-pipeline]] · [[32-static-dynamic-dispatch]] · [[07-glossary]]
