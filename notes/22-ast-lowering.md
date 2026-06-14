# AST lowering (AST -> HIR)

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_ast_lowering`
- **Date:** 2026-06-15

## Question

What does `rustc_ast_lowering` actually do when it turns the AST into HIR — beyond a 1:1 copy — and how does it assign `HirId`s, build the owner structure, handle lifetimes/generics, and desugar surface syntax like `for`, `?`, `if let`, `async`/`await`, and ranges?

## Short answer

Lowering runs after parsing + macro expansion + name resolution and before type checking; it is invoked through the `hir_crate` query (`lower_to_hir`, `compiler/rustc_ast_lowering/src/lib.rs:539`). The crate doc calls it "mostly a simple procedure, much like a fold" (`lib.rs:1`), but the interesting work is *desugaring*: surface constructs are rewritten into a small set of core HIR forms — `for` becomes a `match` + `loop` over `IntoIterator`/`Iterator::next`, `?` becomes a `match` on `Try::branch`, `while` becomes `loop { if ... else break }`, `async`/`await` become coroutine closures and `Poll` loops, and ranges become `Range*` struct literals. Each lowered HIR node gets a `HirId` made of an owner (`OwnerId`) plus a per-owner `ItemLocalId`, via `lower_node_id`/`next_id` (`lib.rs:817`, `:846`). All HIR nodes are allocated into the typed `hir::Arena` (`lib.rs:108`), and each "owner" (item, impl/trait/foreign item, the crate) produces its own indexed `OwnerInfo`.

## Where it sits in the pipeline

`provide` wires `providers.hir_crate = lower_to_hir` (`lib.rs:96`). `lower_to_hir` forces a few prerequisite queries, then *steals* the resolver output and expanded AST produced by resolution:

```rust
let (resolver, krate) = tcx.resolver_for_lowering().steal();   // lib.rs:545
```

`resolver_for_lowering` is built earlier in the interface after name resolution (`compiler/rustc_interface/src/passes.rs:786`). Ordering: parse → expand → resolve (producing `ResolverAstLowering` with res/def-id maps, lifetime resolutions, trait maps) → **lower** → HIR consumed downstream by typeck. NodeIds are assigned to AST nodes *just before* lowering (`lib.rs:10`).

## The `LoweringContext` and the owner structure

`LoweringContext<'a, 'hir>` (`lib.rs:102`) holds per-owner mutable state: `tcx`, a borrow of `ResolverAstLowering`, the HIR `arena`, the accumulated `bodies`/`attrs`/`children`, the current coroutine kind and `task_context` (for `async`), and the `HirId` counters.

Lowering is organized per *owner*. `ItemLowerer` (`item.rs:53`) walks an `ast_index` of all owners and calls `lower_node` for each (`item.rs:98`). For each owner, `with_lctx` creates a *fresh* `LoweringContext` and runs `with_hir_id_owner` (`item.rs:80`). `with_hir_id_owner` (`lib.rs:709`) resets the local-id counter, pre-assigns `ItemLocalId::ZERO` to the owner node (`lib.rs:749`), runs the closure to build a `hir::OwnerNode`, then calls `make_owner_info`. Owner kinds: `Crate`, `Item`, `AssocItem`, `ForeignItem` (`item.rs:104`).

`make_owner_info` (`lib.rs:782`) collects bodies/attrs/trait-map, optionally hashes, then calls `index::index_hir` to build the flat `nodes` array + `parenting` map, and allocates the final `OwnerInfo` into the arena (`lib.rs:804`). The whole crate becomes an `IndexVec<LocalDefId, MaybeOwner>` (`lib.rs:548`, `mid_hir::Crate::new` `:575`).

## Assigning `HirId`s

A `HirId` is `{ owner: OwnerId, local_id: ItemLocalId }`. Two minting functions:

- `lower_node_id(ast_node_id)` (`lib.rs:817`) — for HIR nodes corresponding to an AST node. Uses the current owner + `item_local_id_counter`, increments it, builds the `HirId`. Records `LocalDefId` children, propagates trait-map entries, and (under `debug_assertions`) asserts the same `NodeId` is never lowered twice (`lib.rs:834`).
- `next_id()` (`lib.rs:846`) — for *synthetic* HIR nodes from desugaring with no backing AST `NodeId` (e.g. the `match`/`loop` a `for` expands to). Same counter, no `NodeId` bookkeeping.

Invariant (`lib.rs:15`): every node needs a unique id; reuse an AST id in at most one HIR node, `next_id()` for newly created nodes. `HirIdValidator` later checks local ids were used contiguously.

## Allocating into the HIR arena

`LoweringContext.arena` is the typed `hir::Arena<'hir>` (`lib.rs:108`), declared via `arena_types!(rustc_arena::declare_arena)` in `compiler/rustc_hir/src/lib.rs:49` (list in `arena.rs:4`) and stored on `TyCtxt` as `hir_arena` (`compiler/rustc_middle/src/ty/context.rs:708`). Nodes are interned by `self.arena.alloc(...)` / `alloc_from_iter`, or via the `arena_vec!` helper (`lib.rs:77`). See [[21-arena]].

## Lifetimes and generics

Resolution already resolved lifetimes (incl. elision) into `LifetimeRes`; lowering reads those rather than recomputing. `new_named_lifetime` (`lib.rs:2036`) maps a `LifetimeRes` to a `hir::LifetimeKind`: `Param`→`Param`, `Infer` (`'_`)→`Infer`, `Static`→`Static`, `Fresh`→`Param` of a freshly-created def (`lib.rs:2044`). Elision can introduce *extra* params on a binder; `lower_lifetime_binder` (`lib.rs:987`) pulls them from `resolver.extra_lifetime_params(binder)` and creates them *before* the explicit ones (`lib.rs:992`). `elided_dyn_bound` (`lib.rs:3031`) synthesizes the lifetime for an elided trait-object bound.

## Desugaring example 1: the `for` loop

`lower_expr_for` (`expr.rs:1726`) rewrites `for <pat> in <head> <body>`:

```rust
// before
'a: for x in iter_expr { body }

// after (HIR, pseudo-rust)
{
    let result = match IntoIterator::into_iter(iter_expr) {
        mut iter => 'a: loop {
            match Iterator::next(&mut iter) {
                None => break,
                Some(x) => { body },
            };
        }
    };
    result
}
```

In code: the `None => break` / `Some(<pat>) => <body>` arms (`expr.rs:1751`); `Iterator::next(&mut iter)` via `LangItem::IteratorNext` (`expr.rs:1777`); inner `loop` with `LoopSource::ForLoop` (`expr.rs:1814`); the `IntoIterator::into_iter` match (`expr.rs:1828`); the `drop_temps` wrapper giving `{ let result = ...; result }` semantics (`expr.rs:1879`). Marked `DesugaringKind::ForLoop` (`expr.rs:1737`). `for await` uses `IntoAsyncIterIntoIter` + `make_lowered_await` (`expr.rs:1842`).

## Desugaring example 2: the `?` operator

`lower_expr_try` (`expr.rs:1902`) rewrites `<expr>?`:

```rust
match Try::branch(<expr>) {
    ControlFlow::Continue(val) => #[allow(unreachable_code)] val,
    ControlFlow::Break(residual) =>
        // inside an enclosing `try {}`: break 'catch_target Residual::into_try_type(residual)
        // otherwise:                    return Try::from_residual(residual),
}
```

Scrutinee `Try::branch` via `LangItem::TryTraitBranch` (`expr.rs:1915`); `Continue(val)` arm (`expr.rs:1929`); `Break(residual)` arm chooses `return`/`break` by `try_block_scope` (`expr.rs:1942`); result is a `Match` with `MatchSource::TryDesugar` (`expr.rs:1982`), marked `DesugaringKind::QuestionMark`.

## Other desugarings (briefly)

- **`if let`/`while let`**: NOT fully desugared here — the `let` becomes `hir::ExprKind::Let` (`expr.rs:273`); actual match-lowering happens later in THIR/MIR building.
- **`while`**: `lower_expr_while_in_loop_scope` (`expr.rs:659`) → `loop { if { let _t = cond; _t } { body } else { break } }`, `LoopSource::While`.
- **`if`**: `lower_expr_if` (`expr.rs:624`) → `hir::ExprKind::If` (largely 1:1).
- **`async`/`await`**: `async` blocks/fns → a coroutine *closure* (`expr.rs:887`) carrying a `task_context`; `await` → `make_lowered_await` (`expr.rs:957`) builds a `loop` matching `Poll::Ready`/`Pending`, `DesugaringKind::Await`.
- **ranges**: `lower_expr_range` (`expr.rs:1449`) → struct literal of the matching `Range*` lang item; closed `a..=b` → `RangeInclusive::new` (`expr.rs:1441`).
- **closures**: `lower_expr_closure_expr` (`expr.rs:198`) → `hir::Closure` with its own `def_id`, capture clause, `fn_decl`, lowered body.

## See also

- [[09-source-to-ast]] · [[08-hir-and-mir]] · [[19-hir-format]] · [[21-arena]] · [[11-editions]] · [[07-glossary]]
