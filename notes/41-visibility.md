# Module visibility & item privacy

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_middle/src/ty` (Visibility), `compiler/rustc_resolve`, `compiler/rustc_privacy`
- **Date:** 2026-06-15

## Question

How does Rust control module visibility / item privacy, and how is that modeled, resolved, and enforced inside rustc?

## Short answer

Every item has a *visibility* defaulting to private (visible only in its enclosing module) unless annotated `pub` or a restricted `pub(...)` form. The compiler models this as a two-variant enum `ty::Visibility` — `Public` or `Restricted(DefId)` (the module it's restricted to). `rustc_resolve` turns the surface `ast::Visibility` into a `ty::Visibility` while building the module graph, and computes *effective* visibilities (reach through `pub use`). `rustc_privacy` enforces (1) **access checking** (every item you name is `is_accessible_from` your module) and (2) **private-in-public** (types in a public signature aren't more private than the item — `E0446` + `private_interfaces`/`private_bounds` lints). The rule is recursive: an item is reachable only if it *and every module on the path* are visible from the use site.

## Surface forms & the accessibility rule

```rust
mod outer {
    struct Priv;              // private: only `outer` (+ children)
    pub struct Pub;           // everywhere
    pub(crate) struct Crate;  // anywhere in this crate
    pub(super) struct Super;  // outer's parent module
    pub(self) struct SelfV;   // == private
    pub mod inner { pub(in crate::outer) struct InPath; } // within crate::outer
    pub use inner::InPath as Reexported;  // re-export
}
```

The rule is path-relative: reachable iff the item is visible there **and** every module along the path is too. Encoded by `Visibility::is_accessible_from` (`rustc_middle/src/ty/mod.rs:431`):

```rust
pub fn is_accessible_from(self, module: impl Into<DefId>, tcx: TyCtxt<'_>) -> bool {
    match self {
        Visibility::Public => true,
        Visibility::Restricted(id) => tcx.is_descendant_of(module.into(), id.into()),
    }
}
```

`is_descendant_of` (`ty/mod.rs:404`) walks the def-id tree — "descendant of X" implements `pub(in X)`, `pub(crate)` (X = crate root), and plain private (X = parent module).

## `ty::Visibility` — the representation

Deliberately tiny (`rustc_middle/src/ty/mod.rs:312`):

```rust
pub enum Visibility<Id = LocalDefId> {
    Public,            // visible everywhere, incl. other crates
    Restricted(Id),    // visible only in the given crate-local module
}
```

Every surface form except `pub` collapses to `Restricted(module_def_id)` — no separate `Crate`/`Super`/`InPath` variant. `pub(crate)` = `Restricted(CRATE_DEF_ID)`; private = `Restricted(parent module)`. `to_string` (`:320`) reconstructs the user-facing spelling; `greater_than` (`:459`) orders visibilities by the def-id tree — the backbone of private-in-public.

## Where it's resolved — `rustc_resolve`

`try_resolve_visibility` maps `ast::VisibilityKind` → `ty::Visibility` while building the module graph (`rustc_resolve/src/build_reduced_graph.rs:245`): `Public` → `Public`; `Inherited` → nearest parent `mod` (enum/trait members inherit the enum/trait's visibility); `Restricted{path}` → resolve to a module then `Restricted(def_id)`, verifying the path is an *ancestor* (else `AncestorOnly` error, `:316`). Separately, `compute_effective_visibilities` (`effective_visibilities.rs:80`) computes the maximum reach gained through `pub use` re-exports — what makes `pub use private_mod::Thing` actually expose `Thing`.

## Where it's enforced — `rustc_privacy`

`rustc_privacy::provide` registers `check_mod_privacy` and `check_private_in_public` (`lib.rs:1740`).

**1. Access checking** (`TypePrivacyVisitor`, `lib.rs:1129`): for each item/type named at a use site, `item_is_accessible` checks `tcx.visibility(did).is_accessible_from(self.module_def_id, tcx)` (`:1137`); failure emits `ItemIsPrivate`. `NamePrivacyVisitor` (`:921`) does the same for struct *field* names.

**2. Private-in-public** (`SearchInterfaceForPrivateItemsVisitor`, `:1356`, driven by `check_private_in_public`, `:1884`): walks a public item's interface — `generics`/`predicates`/`bounds`/`ty`/`trait_ref` — and checks each referenced item is at least as visible. Core comparison (`:1415`): if `required_visibility.greater_than(vis, tcx)` → emit `InPublicInterface` (**E0446**, `diagnostics.rs:70`). When the leak is only "more reachable than nominal", it fires the `PRIVATE_INTERFACES`/`PRIVATE_BOUNDS` lints instead (`:1466`). A related `EXPORTED_PRIVATE_DEPENDENCIES` lint fires when a public signature names a type from a private dependency.

Minimal E0446:

```rust
struct Priv;
pub fn leak() -> Priv { Priv }  // error[E0446]: private type `Priv` in public interface
```

`leak` is `Public` but `Priv` is `Restricted(this module)`; `Public.greater_than(Restricted)` is true → error.

## Interaction with codegen symbol visibility

Language-level `ty::Visibility` is *separate* from the linker/symbol visibility (default/hidden) chosen during codegen/monomorphization. A source-private item can still need an externally-visible symbol if reachable through generics/inlining; a `pub` item may get a hidden symbol if not exported. Privacy governs name resolution and the type system, not the symbol table. See [[18-monomorphization]], [[30-abi-status]].

## See also

- [[01-repo-layout]] · [[18-monomorphization]] · [[07-glossary]]
