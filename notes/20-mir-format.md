# The MIR data format

- **Upstream commit:** `4f8ea98e`
- **Area:** `compiler/rustc_middle/src/mir`
- **Date:** 2026-06-15

## Question

How is MIR represented as data in `rustc_middle`? What are the core types — `Body`, basic blocks, statements, terminators, places, rvalues, operands — and how do they fit together for a tiny function?

## Short answer

MIR is a control-flow graph stored in a `Body<'tcx>` (`compiler/rustc_middle/src/mir/mod.rs:206`). The graph nodes are `BasicBlockData` (`mod.rs:1319`), each a straight-line list of `Statement`s ending in exactly one `Terminator` that names its successor blocks. The data flowing through is built from `Place`s (a `Local` plus a list of `ProjectionElem`s), `Operand`s (`Copy`/`Move` of a place, or a `Constant`), and `Rvalue`s (the right-hand side of assignments). Locals are numbered: `_0` is the return value, `_1..=arg_count` are the arguments, the rest are user variables and temporaries (`mod.rs:240`, `mod.rs:867`). A `Body` also records its `MirPhase` — `Built`, `Analysis`, or `Runtime` — which fixes which constructs are legal (`syntax.rs:42`).

## `Body`: the container

`Body<'tcx>` (`mir/mod.rs:206`) holds everything for one function/const body:

- `basic_blocks: BasicBlocks<'tcx>` (`:209`) — the CFG, indexed by `BasicBlock`.
- `phase: MirPhase` (`:216`) — how far through lowering.
- `local_decls: IndexVec<Local, LocalDecl<'tcx>>` (`:240`) — one decl per local. Layout per the doc: return-value pointer first, then `arg_count` argument locals, then user variables and temporaries.
- `arg_count: usize` (`:251`).
- `source_scopes` (`:225`) and `var_debug_info` (`:260`) — debuginfo.

`BasicBlocks` derefs to `IndexVec<BasicBlock, BasicBlockData>` and implements the graph traits (`DirectedGraph`, `StartNode`, `Successors`, `Predecessors`) so dataflow can traverse it (`mir/basic_blocks.rs:112`).

## `BasicBlock` / `BasicBlockData`

`BasicBlock` is a newtype index with `debug_format = "bb{}"` and `START_BLOCK = 0` — blocks print `bb0`, `bb1`, …, execution begins at `bb0` (`mod.rs:1296`).

```rust
pub struct BasicBlockData<'tcx> {
    pub statements: Vec<Statement<'tcx>>,
    pub after_last_stmt_debuginfos: StmtDebugInfos<'tcx>,
    pub terminator: Option<Terminator<'tcx>>,
    pub is_cleanup: bool,
}
```
(`mod.rs:1319`). A block is a flat list of statements with no internal control flow, plus exactly one terminator (`Option` only transiently; `terminator()` `expect`s it, `:1366`). `is_cleanup` marks the unwind path.

## `Statement` / `StatementKind`

`Statement` (`mir/statement.rs:17`) wraps a `SourceInfo` + a `StatementKind` (`syntax.rs`):

```rust
pub enum StatementKind<'tcx> {
    Assign(Box<(Place, Rvalue)>),       // syntax.rs:338  — the workhorse
    FakeRead(..),                       // syntax.rs:355
    SetDiscriminant { place, variant }, // syntax.rs:362
    StorageLive(Local),                 // syntax.rs:382
    StorageDead(Local),                 // syntax.rs:385
    ...
}
```

`Assign` evaluates LHS to a place and RHS to a value, then stores it. `StorageLive`/`StorageDead` mark a local's live range (like stack alloc/dealloc). Statements never branch.

## `Terminator` / `TerminatorKind`

`Terminator` (`mir/terminator.rs:413`) = `SourceInfo` + `TerminatorKind` (`syntax.rs:692`):

- `Goto { target }` — one successor (`:694`).
- `SwitchInt { discr, targets }` — branch on an integer/bool/char to the matching or `otherwise` target (`:704`).
- `Return` — returns; assigns the value in `_0` to the caller (`:724`).
- `Unreachable` — UB if executed (`:740`).
- `Drop { place, target, unwind, .. }` — drop glue (`:774`).
- `Call { func, args, destination, target, unwind, .. }` — call, write result to `destination`, continue at `target` (`:793`).
- Plus `UnwindResume`, `UnwindTerminate`, `Yield`, `FalseEdge`, `FalseUnwind`.

Successor edges come from `successors()` (`terminator.rs:420`), making `BasicBlocks` a graph.

## `Place` / `Local` / `ProjectionElem`

A `Place` is *where* a value lives — an lvalue (`syntax.rs:1176`):

```rust
pub struct Place<'tcx> { pub local: Local, pub projection: &'tcx List<PlaceElem<'tcx>> }
```

`Local` is a newtype index with `debug_format = "_{}"` and `RETURN_PLACE = 0` (`mod.rs:861`); `Local::arg(i)` = `Local(i+1)` (`mod.rs:871`). `ProjectionElem` (`syntax.rs:1185`), with `PlaceElem = ProjectionElem<Local, Ty>` (`:1276`):

- `Deref` (`:1186`), `Field(FieldIdx, T)` (`:1194`), `Index(Local)` (`:1209`), `ConstantIndex`/`Subslice` (`:1226`/`:1247`), `Downcast(_, VariantIdx)` (`:1261`), `OpaqueCast`, `UnwrapUnsafeBinder`.

So `(*_1).field` is `Place { local: _1, projection: [Deref, Field(0, T)] }`.

## `Rvalue` and `Operand`

An `Operand` is a value already in hand (`syntax.rs:1296`):

```rust
pub enum Operand<'tcx> {
    Copy(Place),                  // load; pre-drop-elab the type must be Copy
    Move(Place),                  // load, may leave source uninit (matters for Call args)
    Constant(Box<ConstOperand>),
    RuntimeChecks(RuntimeChecks),
}
```

An `Rvalue` (`syntax.rs:1356`) computes the RHS of an `Assign`:

- `Use(Operand)` (`:1358`), `Repeat(Operand, Const)` (`:1363`), `Ref(Region, BorrowKind, Place)` (`:1372`), `Cast(CastKind, Operand, Ty)` (`:1401`), `BinaryOp(BinOp, Box<(Operand, Operand)>)` (`:1417`), `UnaryOp` (`:1424`), `Aggregate(Box<AggregateKind>, IndexVec<FieldIdx, Operand>)` (`:1446`), `Discriminant(Place)`, `CopyForDeref(Place)`.

`BinOp` distinguishes `Add` from `AddUnchecked` (UB on overflow) and `AddWithOverflow` (returns `(T, bool)`) (`syntax.rs:1614`).

## `MirPhase`: Built → Analysis → Runtime

`MirPhase` (`syntax.rs:42`) records the flavor; each phase restricts what's well-formed:

- `Built` (`:50`) — fresh; few restrictions; consumed by unsafeck, MIR lints, const qualifs.
- `Analysis(AnalysisPhase)` (`:63`) — borrowck. Sub-phases `Initial`/`PostCleanup` (`:100`); later, `FalseEdge`/`FakeRead`/etc. are disallowed.
- `Runtime(RuntimePhase)` (`:94`) — CTFE/opt/codegen. Sub-phases `Initial`/`PostCleanup`/`Optimized` (`:122`). Drops become unconditional, `Yield` gone, aggregates mostly deaggregated.

Differences are *semantic*: e.g. a `Drop` is conditional in Analysis MIR but unconditional in Runtime MIR.

## `SourceScope` / debuginfo (briefly)

Every statement/terminator carries `SourceInfo { span, scope }` (`mod.rs:842`). `scope` indexes `Body::source_scopes`, a tree of `SourceScopeData` (`mod.rs:1478`) tracking visible bindings, lint levels, and inlining provenance. `Body::var_debug_info` (`mod.rs:260`) maps user names to locals/places.

## Worked example: `fn double(x: i32) -> i32 { x + x }`

```text
fn double(_1: i32) -> i32 {
    debug x => _1;                 // VarDebugInfo: name `x` -> Local _1
    let mut _0: i32;               // local_decls[0] = return place (RETURN_PLACE)
    let mut _2: (i32, bool);       // temporary holding (sum, overflowed)

    bb0: {
        _2 = AddWithOverflow(copy _1, copy _1);   // Assign(_2, BinaryOp(AddWithOverflow, (Copy(_1), Copy(_1))))
        assert(!move (_2.1: bool), "attempt to add with overflow", copy _1, copy _1) -> [success: bb1, ...];
    }
    bb1: {
        _0 = move (_2.0: i32);     // Assign(_0, Use(Move(Place{_2,[Field(0,i32)]})))
        return;                    // TerminatorKind::Return — hands _0 back
    }
}
```

Mapping back: `_1` is `local_decls[1]` (`arg_count == 1`); `_0` is `RETURN_PLACE`; the add line is `StatementKind::Assign` with `Rvalue::BinaryOp(AddWithOverflow, ..)`; `(_2.1: bool)` is `Place{_2,[Field(1,bool)]}`; `bb0` is `START_BLOCK`; each block ends in a terminator. A release/unchecked lowering would use plain `BinOp::Add`.

See it: `rustc --emit=mir foo.rs` or `rustc -Z unpretty=mir-cfg foo.rs`.

## See also

- [[08-hir-and-mir]] · [[19-hir-format]] · [[13-mir-optimizations-catalog]] · [[25-ssa]] · [[07-glossary]]
