# Who designs Rust's docs sites & the Playground

- **Upstream commit:** `4f8ea98e`
- **Area:** docs/playground (non-core; rustdoc + external repos)
- **Date:** 2026-06-15

## Question

Who designs and maintains Rust's documentation sites (rustdoc output, docs.rs, rust-lang.org, The Book) and the Rust Playground — which teams, which repos, who did the visual/UI design?

## Short answer

There is no single "Rust design team." The work is split across teams and repos. **rustdoc's** HTML/CSS/JS theme lives in-tree (`src/librustdoc/html/`) and is owned by the **rustdoc team** (`T-rustdoc`, with a frontend autolabel `T-rustdoc-frontend`). **docs.rs** (crate-doc hosting) is a separate repo/team (`rust-lang/docs.rs`). The **marketing site** rust-lang.org is another repo (`rust-lang/www.rust-lang.org`). The **Playground** is its own repo (`rust-lang/rust-playground`), a React + Axum app associated with the Infrastructure team and hosted by Integer 32. Visual/branding work is mostly community/in-project; the Rust Foundation owns the trademarks/logos. No dedicated design agency was found for any of these.

## rustdoc & the docs theme (repo-grounded)

The most directly verifiable part. rustdoc is in-tree: `src/librustdoc/` (HTML generation, search), `src/tools/rustdoc/` (thin wrapper), and theme assets under `src/librustdoc/html/static/` (`css/` with `rustdoc.css`/`normalize.css`, `js/`, `fonts/`, `images/`). Ownership in `triagebot.toml`:

- `[autolabel."T-rustdoc"]` — `src/librustdoc`, `src/tools/rustdoc`, `src/rustdoc-json-types`, `src/doc/rustdoc/`.
- `[autolabel."T-rustdoc-frontend"]` — specifically the web frontend: `src/librustdoc/html/`, `tests/rustdoc-gui/`, `tests/rustdoc-js/`, areas `A-rustdoc-search`/`-ui`/`-js`. A comment notes `tests/rustdoc-ui` tests the CLI, not the web frontend — the project deliberately separates rustdoc's CLI/backend from its HTML/CSS/JS frontend.

> Uncertainty: `T-rustdoc-frontend` is a triagebot *label* for routing PRs; the chartered governance unit is a single **rustdoc** team (a devtools subteam), not necessarily a formal "frontend" subteam. The theme was redesigned within the project over the years (frontend work publicly associated with maintainers like notriddle / GuillaumeGomez), not by an outside firm. Current roster: `rust-lang/team/teams/rustdoc.toml`.

## docs.rs (web-sourced)

Repo `rust-lang/docs.rs` (formerly "cratesfyi"); auto-builds & hosts docs for crates.io crates using rustdoc under nightly. Maintained by the docs.rs team (`#t-docs-rs`). Stack: a Rust service + PostgreSQL + S3 + Docker build isolation. Its doc pages inherit rustdoc's theme; docs.rs adds its own surrounding chrome/navigation.

## The website rust-lang.org (web-sourced)

Repo `rust-lang/www.rust-lang.org` (old site archived at `rust-lang/prev.rust-lang.org`); auto-deployed via GitHub Pages. This is the marketing/landing site with its **own distinct design**, separate from rustdoc's doc theme; community-driven with Foundation/marketing involvement, translations via Pontoon (`rust-www`). No single dedicated "website design" owner pinned down — uncertain.

The Book and other long-form docs (TRPL, Rust by Example, the Reference, rustc-dev-guide) are separate mdBook-based repos (e.g. `rust-lang/book`), each with their own styling.

## The Playground (web-sourced)

Repo `rust-lang/rust-playground`, deployed at play.rust-lang.org. Per its README: "A React frontend communicates with an Axum backend. Docker containers are used to provide the various compilers and tools" (Rust ~52%, TypeScript ~35%). Also "hosted by **Integer 32**." Generally community-maintained infra (Infrastructure team); the README names no owning team and **credits no UI/visual designer** — the UI appears project/community-designed, not agency work (stated honestly; not documented anywhere found).

## Branding / Rust Foundation

The **Rust Foundation** owns and stewards the **Rust and Cargo trademarks and logos** (transferred from Mozilla at founding); marks include Rust, Cargo, Clippy, the logo, and the site's styling. Logos are CC-BY but use is governed by trademark law (logo/media policy). The Foundation can fund infra/branding work, but no specific record attributes the rustdoc theme, website design, or playground UI to Foundation-commissioned design — uncertain.

**Bottom line:** four different things, four repos/owners — (1) rustdoc theme = in-tree `src/librustdoc/html/`, rustdoc team; (2) marketing site = `rust-lang/www.rust-lang.org`; (3) playground UI = `rust-lang/rust-playground`; (4) docs.rs = `rust-lang/docs.rs`, layered on rustdoc output.

## See also

- [[33-governance-participation]] · [[01-repo-layout]] · [[07-glossary]]

Sources: github.com/rust-lang/{rust-playground,docs.rs,www.rust-lang.org,team,book} · play.rust-lang.org · rust-lang.org/governance · foundation.rust-lang.org logo policy · rust-lang.github.io rustdoc-team-charter
