# MIR optimizations & transforms — a visual catalog

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_mir_transform`
- **Date:** 2026-06-14

## Question

What optimizations/transforms happen on the way to (and within) MIR? As many as
possible, with before→after pseudocode.

## Notation

`_0` = return value, `_1, _2, ...` = locals, `bbN` = basic blocks. Examples are
simplified MIR-ish pseudocode. Most opt passes only run at `-C opt-level >= 1`
(those wrapped in `o1(...)` = `WithMinOptLevel(1, _)`); heavy low-level work
(loop unrolling, vectorization) is left to LLVM — see [[12-optimization-pipeline]].

## Part 0 — transforms while *building* MIR

### Constant promotion (`promote_consts.rs`)

Borrows of constant rvalues are promoted to separate `'static` constants.

```rust
let r: &i32 = &5;
// -> promoted[0] = const 5
//    _r = &promoted[0]      // 'static
```

### Drop elaboration (`elaborate_drops.rs`)

Implicit end-of-scope drops become explicit drop terminators, with **drop flags**
when a value may have been moved out conditionally.

```rust
{ let s = String::from("hi"); if cond { consume(s); } }   // implicit drop
// ->
//   drop_flag = true
//   if cond { consume(s); drop_flag = false }
//   if drop_flag { drop(s) }     // avoids double drop
```

### Lowering builtins (`lower_slice_len.rs`, `lower_intrinsics.rs`)

```rust
s.len()           // -> PtrMetadata(s)   (lower_slice_len; runs before inlining)
intrinsics::add() // -> an actual Add rvalue, etc. (lower_intrinsics)
```

## Part 1 — MIR optimization passes (`run_optimization_passes`)

> UB checks are inserted **first** (`CheckAlignment`, `CheckNull`, `CheckEnums`)
> so optimizations can't delete the UB before it's checked.

### InstSimplify — peephole (`instsimplify.rs`)

Rules are listed verbatim in the source:

```text
Eq(a, true)  => a       Ne(a, false) => a
Eq(a, false) => !a      Ne(a, true)  => !a
&(*a)        => a                 // ref-of-deref cancels
[x; 1] element read              // repeated-aggregate simplification
i32 as u32 (same-size int) => signedness cast (effectively nop)
```

### Inline (`inline.rs`)

```text
_2 = add(_1, const 1)        =>    _2 = Add(_1, const 1)   // callee body spliced in
```

### GVN — global value numbering = CSE + const folding (`gvn.rs`)

> "MIR may contain repeated and/or redundant computations ... re-use the
> already-computed result when possible."

```text
_3 = Add(_1, _2)                   _3 = Add(_1, _2)
_4 = Add(_1, _2)            =>      _5 = Mul(_3, _3)    // _4 ≡ _3
_5 = Mul(_3, _4)
```

### DataflowConstProp — constant propagation (`dataflow_const_prop.rs`)

```text
_1 = const 1
_2 = Add(_1, const 2)       =>      _2 = const 3
```

### ReferencePropagation (`ref_prop.rs`)

```text
_2 = &_1
_3 = (*_2)                  =>      _3 = _1
```

### CopyProp (`copy_prop.rs`)

```text
_2 = _1
_3 = Add(_2, const 1)       =>      _3 = Add(_1, const 1)
```

### DeadStoreElimination (`dead_store_elimination.rs`)

```text
_2 = const 5     // never read
_3 = foo()                  =>      _3 = foo()
```

### DestinationPropagation (`dest_prop.rs`)

> "MIR building can insert a lot of redundant copies ... LLVM by itself is not
> good enough at eliminating these."

```text
_3 = move _2
_0 = move _3                =>      // use _2 directly as _0; drop _3
```

### SROA — scalar replacement of aggregates (`sroa.rs`)

```text
_1 = Point { x: _a, y: _b }        _1_x = _a
_2 = (_1.x)                 =>      _1_y = _b
                                   _2   = _1_x
```

### JumpThreading (`jump_threading.rs`) — figure from the source

```text
   X = 0                          X = 0
------------\     /--------     ------------
   X = 1     X---X Switch(X)  =>     X = 1
------------/     \--------     ------------
(join-then-switch where the value is known => jump straight)
```

### MatchBranchSimplification (`match_branches.rs`)

```rust
match b { false => _2 = 0, true => _2 = 1 }   =>   _2 = b as u8  // switch -> arithmetic
```

### EarlyOtherwiseBranch (`early_otherwise_branch.rs`)

```rust
match (a, b) { (Some(_), Some(_)) => .., _ => .. }
// merges repeated discriminant reads of the same value into one
```

### UnreachableEnumBranching / UnreachablePropagation (`unreachable_prop.rs`)

```text
switch(discr) { 0 => A, 1 => B(uninhabited) }  =>  switch(discr) { 0 => A }
// then propagate `unreachable` back to predecessors whose successors are all unreachable
```

### RemoveZsts / RemoveUnneededDrops

```text
_2 = ()                      =>   (ZST ops removed; ZST operands -> const)
drop(_1)  // _1 has no Drop   =>   (removed)
```

### SimplifyCfg / SimplifyLocals (`simplify.rs`)

```text
bb0: goto bb1
bb1: goto bb2                =>   merge straight-line blocks; drop empty blocks
bb2: ...                          (SimplifyLocals also removes unused local decls)
```

### Misc

- `SimplifyComparisonIntegral` — simplify integral comparison chains.
- `EnumSizeOpt` / `large_enums` — optimize layout of large enums.
- `MultipleReturnTerminators` — `goto`→return becomes `return`.
- `RemoveStorageMarkers`, `RemoveNoopLandingPads` (drop useless unwind landing
  pads), `StripDebugInfo`, `CriticalCallEdges` (split critical edges for LLVM).

## End-to-end example

```rust
fn f(a: i32) -> i32 {
    let x = a + 1;
    let y = a + 1;   // same as x
    let z = x + y;
    z
}
```

```text
GVN:          y ≡ x          => z = x + x
ConstProp:    (if a const)   => fold everything
CopyProp+DSE: clean temps x, y, z
Result:       roughly `_0 = a + a` ; several blocks collapse to one
```

## See also

- [[12-optimization-pipeline]] — full pipeline + where LLVM takes over
- [[08-hir-and-mir]] — what MIR is
- MIR opt guide: https://rustc-dev-guide.rust-lang.org/mir/optimizations.html
