# The HIR data format

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_hir`
- **Date:** 2026-06-15

## Question

How is the High-level Intermediate Representation (HIR) structured in `compiler/rustc_hir/src/hir.rs` — its top-level containers, node kinds, identifiers, owner-indexing, arena allocation, and how it differs from the AST?

## Short answer

HIR is the desugared, name-resolved tree the compiler builds by lowering the AST. It is *owner-indexed*: the crate is a flat `IndexVec` of "owners" (item-likes), each holding a dense `IndexVec<ItemLocalId, ParentedNode>` of its inner nodes. Every node is addressed by a two-part `HirId = OwnerId + ItemLocalId`, which keeps ids stable under edits for incremental compilation. Nodes are bump-allocated in an arena and wired together with `&'hir` references rather than `Box`. Unlike the AST, HIR has no types attached inline — types come later from typeck (`TypeckResults`, keyed by `HirId`) — and many surface constructs (`for`, `while let`, `?`, `.await`) are desugared away.

## Top level: `Crate` and `OwnerNodes`

The whole crate is `rustc_middle::hir::Crate` (not in `rustc_hir` itself), a flat index from `LocalDefId` to a `MaybeOwner` (`compiler/rustc_middle/src/hir/mod.rs:36`). A `MaybeOwner` is either an actual owner, a back-pointer for non-owners, or a placeholder (`compiler/rustc_hir/src/hir.rs:1749`):

```rust
pub enum MaybeOwner<'tcx> {
    Owner(&'tcx OwnerInfo<'tcx>),
    NonOwner(HirId),
    Phantom,
}
```

Each owner's lowered contents live in `OwnerInfo` (`hir.rs:1721`), whose `nodes` field is the heart of the tree, `OwnerNodes` (`hir.rs:1664`):

```rust
pub struct OwnerNodes<'tcx> {
    pub opt_hash_including_bodies: Option<Fingerprint>,
    pub nodes: IndexVec<ItemLocalId, ParentedNode<'tcx>>,
    pub bodies: SortedMap<ItemLocalId, &'tcx Body<'tcx>>,
}
```

So an owner is *not* a recursive tree of boxes — it is a flat vector indexed by `ItemLocalId`, where each slot is a `ParentedNode` (`hir.rs:1268`) pairing a `Node` with its parent's local id within the same owner. Slot zero (`ItemLocalId::ZERO`) is always the owner node itself; its parent is set to `INVALID` so accidental access ICEs (`hir.rs:1678`, `:1690`).

## Identifiers: `HirId = OwnerId + ItemLocalId`

`HirId` lives in its own crate (`compiler/rustc_hir_id/src/lib.rs:86`):

```rust
pub struct HirId { pub owner: OwnerId, pub local_id: ItemLocalId }
```

- `OwnerId` wraps the `LocalDefId` of the enclosing item-like (`hir_id/src/lib.rs:16`).
- `ItemLocalId` is dense within an owner, starting at zero (`hir_id/src/lib.rs:146`) — which is what lets `OwnerNodes::nodes` be an `IndexVec`.

The two-level design is for incremental compilation: moving an item changes its `OwnerId` but leaves every contained `ItemLocalId` unchanged, so results keyed by local id survive edits (`hir_id/src/lib.rs:74`). `HirId` deliberately does **not** implement `Ord` (`:94`). Root constant: `CRATE_HIR_ID` (`:175`).

## `Item` / `ItemKind`

An `Item` is an owner, identified by `owner_id` (not an inline `HirId`) (`hir.rs:4553`); its `hir_id()` = `HirId::make_owner(owner_id.def_id)` (`hir.rs:4566`). `ItemKind` (`hir.rs:4776`) enumerates `ExternCrate`, `Use`, `Static`, `Const`, `Fn { sig, ident, generics, body: BodyId, .. }`, `Mod`, `TyAlias`, `Enum`, `Struct`, `Union`, `Trait`, `Impl`, etc. A function's body is **not** inline — `Fn` holds a `BodyId` (`hir.rs:4794`).

## `Expr` / `ExprKind`

Expressions carry an inline `HirId` (`hir.rs:2553`). `ExprKind` (`hir.rs:2909`) covers `Call`, `MethodCall`, `Binary`, `Lit`, `If`, `Loop`, `Match`, `Block`, `Path(QPath)`, `Struct`, etc. HIR-specific bits:

- `DropTemps(&Expr)` (`hir.rs:2957`) — synthetic, models drop order during desugaring; no AST equivalent.
- Surface loops are gone: `Loop(.., LoopSource, ..)` (`hir.rs:2977`) with `LoopSource::{Loop, While, ForLoop}` (`hir.rs:3167`) — `while`/`for` lower to `loop` + `match`/`if`.
- `MethodCall` stores no callee `DefId` inline — resolve via `type_dependent_def_id` after typeck (`hir.rs:2933`).

## `Stmt` and `Body`

`Stmt` carries an inline `HirId` (`hir.rs:2168`); `StmtKind` (`hir.rs:2177`) is `Let`, `Item(ItemId)`, `Expr(&Expr)`, or `Semi(&Expr)`. A `Body` (`hir.rs:2292`) is `{ params: &'hir [Param], value: &'hir Expr }`, stored on the owner in `OwnerNodes::bodies` keyed by `ItemLocalId` (`hir.rs:1674`) and referenced via `BodyId` (`hir.rs:2266`).

## `Node` and `OwnerNode`

`Node` (`hir.rs:5140`) is the universal "anything in the HIR" enum — `Param`, `Item`, `Expr`, `Stmt`, `Pat`, `Ty`, `Block`, `Crate(&Mod)`, etc. `OwnerNode` (`hir.rs:5006`) is the smaller subset that can be a slot-zero owner.

## Arena allocation (`&'hir`)

Every nested reference is `&'hir T`, not `Box`/`Vec`. HIR is bump-allocated into a typed arena; allocatable types are listed by `arena_types!` (`compiler/rustc_hir/src/arena.rs:4`), and any `Copy` type is allocatable by default. That's why slices appear as `&'hir [Expr<'hir>]` (e.g. `Call`'s args, `hir.rs:2920`) — the whole tree shares one lifetime and is freed at once. See [[21-arena]].

## Navigating: the HIR map

The "HIR map" is methods on `TyCtxt` (`compiler/rustc_middle/src/hir/map.rs`). Core lookup decodes a `HirId` into its owner's vector:

```rust
pub fn hir_node(self, id: HirId) -> Node<'tcx> {
    self.hir_owner_nodes(id.owner).nodes[id.local_id].node
}
```
(`map.rs:164`)

Parent navigation uses `ParentedNode.parent`, with a special case at slot zero crossing owner boundaries via `hir_owner_parent` (`map.rs:178`).

## How HIR differs from the AST

- **Desugared.** `for`/`while let` → `Loop` + `Match`; `?` → `Match` (`MatchSource::TryDesugar`, `hir.rs:3143`); `DropTemps` inserted.
- **Name-resolved paths.** AST paths are unresolved; HIR uses `QPath` (`hir.rs:3072`): `Resolved` carries a resolution, `TypeRelative` (e.g. `<T>::Output`) left for typeck (`hir.rs:3081`).
- **No inline types.** `Expr` has no type field. Types live in `TypeckResults`, keyed per-owner by `ItemLocalId` — `node_types` (`compiler/rustc_middle/src/ty/typeck_results.rs:48`), accessed via `expr_ty` (`:367`). Keying by `ItemLocalId` is exactly why the local id is owner-scoped.
- **Owner-indexed & arena-backed** with stable `HirId`s, vs the AST's `NodeId`-tagged boxed tree.

## A concrete example: `fn main() {}`

One owner. Slot-zero `Node` = `OwnerNode::Item(&Item)` with `owner_id` = `main`'s `LocalDefId` and `kind = ItemKind::Fn { ident: "main", .., body: BodyId, .. }` (`hir.rs:4794`). The `BodyId` points into the owner's `bodies` map at `Body { params: &[], value }`. The `value` is an `Expr` with `kind = ExprKind::Block(&Block, None)` (`hir.rs:2989`); the empty `Block` has no statements/tail.

For `1 + 2` (tail expr): `Expr { kind: Binary(Add, lhs, rhs) }`, where `lhs`/`rhs` are `&'hir Expr` with `ExprKind::Lit(..)`. All three `Expr`s sit in the same owner's `nodes` vector under distinct `ItemLocalId`s, children's `parent` pointing at the `Binary` node. None carries a type until typeck fills `node_types`.

## See also

- [[08-hir-and-mir]] · [[09-source-to-ast]] · [[22-ast-lowering]] · [[20-mir-format]] · [[21-arena]] · [[06-reading-the-compiler]] · [[07-glossary]]
