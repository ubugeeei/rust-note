# From source code to AST (lexing & parsing)

- **Upstream commit:** `4f8ea98e`
- **Area:** `rustc_lexer`, `rustc_parse`, `rustc_ast`
- **Date:** 2026-06-14

## Question

How does raw source text become an AST?

## Short answer

Four stages, with lexing split into **two layers**:
`&str` → **`rustc_lexer`** (pure, reusable raw tokens) → **`rustc_parse::lexer`**
(rustc tokens with spans + interning, grouped into a `TokenStream` of token
trees) → **Parser** (recursive descent) → **AST** (`Crate`). The two-layer lexer
is the distinctive part: `rustc_lexer` knows nothing rustc-specific; the parse
crate's lexer adds spans, `Symbol` interning, and error reporting.

## Pipeline

```
&str (source)
  -> (1) rustc_lexer         : raw token stream (pure, reusable)
  -> (2) rustc_parse::lexer  : wide tokens + TokenStream (spans, interning)
  -> (3) Parser              : recursive descent
  -> (4) AST (Crate)
```

### (1) `rustc_lexer` — pure lexing (`compiler/rustc_lexer/src/lib.rs`)

Its module doc states the design intent:
> operates directly on `&str`, produces simple tokens which are a pair of
> type-tag and a bit of original text, and does not report errors, instead
> storing them as flags on the token. Tokens produced by this lexer are not yet
> ready for parsing.

- Input: plain `&str`. Output: `TokenKind` (type tag + length of original text).
- **No spans, no interning, no error reporting** — errors are flags on tokens.
- Deliberately rustc-agnostic so it's a **reusable library** (usable by rustfmt
  etc.).
- Whitespace and comments are emitted as tokens at this level.

### (2) `rustc_parse::lexer` — promote to rustc tokens (`compiler/rustc_parse/src/lexer/mod.rs`)

Converts the raw stream into the "wide" tokens the real parser consumes:
- Attaches **spans** (`rustc_span` source positions).
- **Interns** identifiers/literals into `Symbol`s (one id per unique string).
- Actually **reports errors**.
- Entry `lex_token_trees` (`lexer/mod.rs:66`) builds the **`TokenStream` /
  `TokenTree`**: matched delimiters `()` `[]` `{}` are grouped together. This
  grouped stream is the unit consumed by both the parser **and** macros.

### (3) Parser — recursive descent (`compiler/rustc_parse/src/parser/`)

- Entry points: `new_parser_from_source_str` / `new_parser_from_file`
  (`compiler/rustc_parse/src/lib.rs:93` / `:108`).
- Split by grammar element: `item.rs` (fns/structs/...), `expr.rs`, `stmt.rs`,
  `pat.rs`, `ty.rs`, `path.rs`, `generics.rs`.
- Consumes the `TokenStream` with lookahead, building AST nodes.

### (4) AST (`compiler/rustc_ast/src/ast.rs`)

- Result is a `Crate` (`ast.rs:550`): a list of `Item`s (`ItemKind` `:4060`),
  each holding the expression/statement tree beneath it.

## Concrete example: `fn main() {}`

| Stage | Roughly |
|-------|---------|
| (1) raw tokens | `Ident("fn")` `Whitespace` `Ident("main")` `OpenParen` `CloseParen` `Whitespace` `OpenBrace` `CloseBrace` |
| (2) wide tokens + TokenStream | `fn` (keyword), `main` (interned `Symbol`, with span), `Delimited(())` empty, `Delimited({})` empty; whitespace dropped |
| (3)->(4) AST | `Crate { items: [ Item { kind: Fn { name: "main", body: <empty block> } } ] }` |

Key rustc-specific point: delimiters `()` `{}` are grouped into **token trees
(matched pairs) before parsing**, because macro expansion operates on the same
`TokenStream`.

## Notes & open questions

- Why two lexer layers? Separation of concerns + reuse: `rustc_lexer` stays pure
  and portable; rustc-specific spans/interning/diagnostics live in `rustc_parse`.
- Follow-up: where macro expansion fits (`rustc_expand`) — it runs on the token
  stream / early AST, before the AST is fully formed. Worth its own note.
- See it: `rustc -Z unpretty=ast-tree` dumps the AST.

## See also

- [[08-hir-and-mir]] — the next lowerings after AST
- [[06-reading-the-compiler]] · [[07-glossary]]
- parsing guide: https://rustc-dev-guide.rust-lang.org/the-parser.html
