# How editions are implemented

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_span` (edition, symbol, hygiene), `rustc_session`
- **Date:** 2026-06-14

## Question

How are editions (2015/2018/2021/2024) actually implemented?

## Short answer

An edition is just an **ordered enum** in `rustc_span`, so checks are plain `>=`
comparisons (`at_least_rust_2018()` etc.). It's a **per-crate** setting stored in
`Session`, but crucially each `Span` also **remembers its own edition** (via
hygiene), which is what makes mixing crates/macros of different editions work.
Behavior is switched in three places: **conditional keywords** in the
lexer/parser, **edition branches** in later passes, and **compatibility lints**
for `cargo fix --edition`. All editions lower to the **same IR/ABI**, so they
interoperate.

## How it works

### 1. Representation: an ordered enum (`compiler/rustc_span/src/edition.rs:9`)

```rust
pub enum Edition { Edition2015, Edition2018, Edition2021, Edition2024, ... }
```

Ordering is `2015 < 2018 < 2021 < 2024`, so feature checks are comparisons
(`edition.rs:94`):

```rust
pub fn at_least_rust_2018(self) -> bool { self >= Edition::Edition2018 }
```

`DEFAULT_EDITION = 2015`, `LATEST_STABLE_EDITION = 2024` (`:50`, `:52`).

### 2. Where the edition comes from

Per-crate via the `--edition` flag, stored in `Session`. Read it via:
- `sess.edition()` — whole-crate (`rustc_session/src/session.rs:902`)
- `span.edition()` — **the edition of that span/token** (`rustc_span/src/lib.rs:995`)
- hygiene (`rustc_span/src/hygiene.rs:912`)

**Key design:** the edition is carried by a `Span`'s hygiene/`SyntaxContext`.
So a macro written in a 2015 crate keeps its 2015 tokens even when expanded
inside a 2021 crate — which is part of why **crates of different editions link
together**.

### 3. Effect #1: edition-conditional keywords (lexer/parser)

The clearest implementation, in `compiler/rustc_span/src/symbol.rs`:

```rust
fn is_used_keyword_conditional(self, edition) -> bool {
    (self >= kw::Async && self <= kw::Dyn) && edition() >= Edition::Edition2018  // :2947
}
fn is_unused_keyword_conditional(self, edition) -> bool {       // :2950
    self == kw::Gen && edition().at_least_rust_2024()           // `gen` reserved in 2024+
    || self == kw::Try && edition().at_least_rust_2018()        // `try` reserved in 2018+
}
pub fn is_reserved(self, edition) -> bool { ... }               // :2955 combines them
```

So `async` / `await` / `dyn` are **ordinary identifiers in 2015, keywords in
2018+**. `is_reserved` takes the edition as a closure, so each token is judged by
its own edition.

### 4. Effect #2: semantic / lowering branches

Later passes branch on `if edition.at_least_rust_2021() { ... }`. Examples:
- 2018: module path resolution changes, `dyn Trait` warnings.
- 2021: disjoint closure captures, array `IntoIterator`, `panic!` macro change.
- 2024: more reserved keywords, default-behavior tweaks.

All of these lower to the **same internal IR / type system / ABI**, which is why
editions don't split the ecosystem.

### 5. Effect #3: migration lints

Each edition has a compatibility lint group (`edition.rs:68`, `lint_name()`):

```
2018 -> "rust_2018_compatibility"
2021 -> "rust_2021_compatibility"
2024 -> "rust_2024_compatibility"
```

These power `cargo fix --edition`, providing the diagnostics that auto-rewrite
old code to the new edition.

## Takeaways

1. Ordered enum → checks are just `>=`.
2. Edition is **remembered per span** (for macros + mixed-edition linking).
3. **Keyword reservation** is switched by edition in the lexer/parser.
4. Passes branch on edition, but everything **lowers to a shared IR/ABI** →
   compatibility + no ecosystem split.
5. **Compatibility lints** drive automated migration.

## Notes & open questions

- Follow-up: how hygiene/`SyntaxContext` stores the edition, and how macro
  expansion threads it through (`rustc_expand`).

## See also

- [[09-source-to-ast]] — keywords are decided during lexing/parsing
- [[10-performance-and-robustness]] — editions as a compatibility mechanism
- [[07-glossary]]
- edition guide: https://doc.rust-lang.org/edition-guide/
