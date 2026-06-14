# HIR & MIR: roles and concrete examples

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_hir`, `rustc_middle::mir`
- **Date:** 2026-06-14

## Question

What are HIR and MIR for, and what do they actually look like?

## Short answer

**HIR** is a **desugared, uniform syntax tree** — still tree-shaped and close to
source, but with sugar (`for`, `?`, `if let`, ...) removed. It's the IR type
checking runs on. **MIR** lowers that further into a **control-flow graph**:
explicit local variables (`_0` = return value), basic blocks, terminators, and
made-explicit operations like drops and overflow checks. MIR is what borrow
checking, const-eval, optimization, and codegen consume.

## Roles

### HIR (High-level IR) — "the tidied-up syntax tree"

`compiler/rustc_hir/src/lib.rs:1` — *"HIR datatypes."*

- Tree-shaped, still recognizably the source (exprs, stmts, items).
- **Desugars** surface syntax into a smaller core (`for`, `?`, `if let`, etc.).
- The IR **type checking** runs on (`rustc_hir_typeck`, `rustc_infer`).
- Every node has a `HirId` for later reference.
- Produced from the AST by `rustc_ast_lowering`. Key types: `Expr`
  (`compiler/rustc_hir/src/hir.rs:2553`), `ExprKind` (`:2909`).

Evidence of desugaring in the source — `ExprKind::DropTemps` doc:
> This construct only exists to tweak the drop order in AST lowering. An example
> of that is the desugaring of `for` loops.

### MIR (Mid-level IR) — "the control-flow graph"

`compiler/rustc_middle/src/mir/mod.rs:1` — *"MIR datatypes and passes."*

- **Not a tree** — a CFG of basic blocks. `BasicBlockData`
  (`mir/mod.rs:1319`), `TerminatorKind` (`mir/syntax.rs:692`).
- **Explicit locals**: `_0` is always the return value, `_1`, `_2`, ... the
  rest. Values are `Rvalue` (`mir/syntax.rs:1356`), assigned into a `Place`.
- **Implicit work made explicit**: drops (destructor calls), overflow checks,
  etc. appear as statements/terminators.
- Consumed by borrow checking (`rustc_borrowck`), const-eval, MIR optimization
  (`rustc_mir_transform`), and codegen. Built by `rustc_mir_build`.

## Concrete examples

### Example 1 — a `for` loop is desugared at HIR level (HIR's job)

Source:

```rust
for x in iter { body() }
```

Becomes, roughly, at HIR:

```rust
match IntoIterator::into_iter(iter) {
    mut it => loop {
        match it.next() {
            Some(x) => { body() }
            None => break,
        }
    },
}
```

So later phases never need to know about `for` — HIR **normalizes sugar into a
few core constructs**.

### Example 2 — arithmetic lowered to a CFG (MIR's job)

Source:

```rust
fn double(x: i32) -> i32 {
    x + x
}
```

Representative MIR (debug build, overflow checks on; exact formatting/numbers
vary by version):

```text
fn double(_1: i32) -> i32 {
    debug x => _1;            // source `x` is local _1
    let mut _0: i32;          // _0 is always the return value
    let mut _2: (i32, bool);  // (sum, overflow flag)

    bb0: {
        _2 = CheckedAdd(_1, _1);
        assert(!move (_2.1: bool),
               "attempt to add with overflow") -> [success: bb1, unwind: continue];
    }
    bb1: {
        _0 = move (_2.0: i32);
        return;
    }
}
```

One line `x + x` becomes two basic blocks with an explicit overflow check —
that explicitness is exactly what makes MIR good for analysis and codegen.

## One-line contrast

- **AST**: as written (syntax).
- **HIR**: desugared, uniform tree → good for **type checking**.
- **MIR**: CFG + explicit locals → good for **borrow checking, optimization,
  codegen**.

## Notes & open questions

- See it yourself once we can build: `rustc -Z unpretty=hir-tree` and
  `rustc -Z unpretty=mir`.
- Follow-up: who builds each? AST→HIR = `rustc_ast_lowering`; HIR→MIR =
  `rustc_mir_build`. Worth a note tracing one function through both.

## See also

- [[07-glossary]] · [[06-reading-the-compiler]]
- HIR guide: https://rustc-dev-guide.rust-lang.org/hir.html
- MIR guide: https://rustc-dev-guide.rust-lang.org/mir/index.html
