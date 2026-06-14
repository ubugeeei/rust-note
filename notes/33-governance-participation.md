# Rust project: governance, leadership & how to participate

- **Upstream commit:** `4f8ea98e`
- **Area:** project governance (non-code)
- **Date:** 2026-06-15

## Question

Who "owns" or leads the Rust project, how is it governed, and how does someone actually participate?

## Short answer

There is no CEO, owner, or single leader: the Rust Project is run collectively by teams (compiler, lang, libs/libs-api, types, infra, release, dev-tools, and more), each responsible for its own domain. Top-level governance is the **Leadership Council** (RFC 3392, June 2023), one representative per top-level team; it replaced the old core team and handles only cross-cutting governance. The **Rust Foundation** is a separate non-profit (board + executive director) handling trademark, funding, and infra — **not** language/technical decisions. Consequential decisions go through the public **RFC process** (`rust-lang/rfcs`); day-to-day work flows through PRs reviewed by team members and merged via the bors/merge-queue gate. You participate by contributing, joining forums (internals, Zulip), writing/commenting on RFCs, and eventually being nominated to a team.

## No single owner: the Project is run by teams

A federation of teams, each with authority over its area. Team boundaries are visible in this repo's `triagebot.toml`, which auto-routes issues/PRs. Labels present include `T-compiler`, `T-libs`, `T-infra`, `T-release`, `T-bootstrap`, `T-style`, `T-rustdoc`, `T-clippy`, `T-rust-analyzer`, `T-rustfmt`, `T-types`. These map to real teams: compiler, lang (design), libs / libs-api (impl vs API/stability), types (formal semantics), infrastructure, release, dev-tools, plus tool teams. (The `triagebot.toml` label set is broader than the formal "top-level team" list used for Council representation.)

## The Leadership Council (RFC 3392, 2023)

- **What:** the top-level governance body, established when **RFC 3392 merged 2023-06-20**.
- **Replaced:** the former **core team** (and the interim post-core leadership chat).
- **Composition:** one representative chosen by each top-level team (nine at founding) — authority delegated *upward* from teams.
- **Scope (important):** only top-level/cross-cutting governance. It does **not** take over technical work — compiler maintenance, language/library evolution, and infra administration stay with the teams.

> Membership rotates per-team, so this note deliberately does not name current members. (Historical only: the June 2023 founding cohort included reps such as Eric Holk, Jack Huey, Mara Bos, Mark Rousskov — a *2023* roster, not current.) Check the live `rust-lang/leadership-council` repo.

## The Rust Foundation — separate non-profit

Independent non-profit, distinct from the Project; a **board of directors** + **executive director**. Role: stewarding the **trademark**, **funding** (grants, sponsorships, security work), and **infrastructure**. It does **not** make language/library/technical design decisions. The bridge: the Project elects **Project Directors** who sit on the Foundation board (defined in `leadership-council/roles/rust-foundation-project-director.md`). The 2024–2025 trademark policy revision illustrates the split — Foundation drafted/approved policy; technical direction stayed with the Project. See [[03-governance-and-business]].

## How decisions get made: the RFC process

Substantial changes go through `rust-lang/rfcs`: open an RFC PR (motivation/design/drawbacks/alternatives) → public discussion → owning team moves it to **FCP (Final Comment Period)** via rfcbot (team sign-off + 10-day window) → merge = accepted, tracking issue opened. Smaller changes skip RFCs and go straight to a PR with team sign-off. Working/project groups form around focused efforts and feed the owning team(s).

## How to participate

From this repo's `CONTRIBUTING.md`:
- **Ask first:** the #new members Zulip stream, broader `rust-lang.zulipchat.com`, and `internals.rust-lang.org`.
- **Read the guides:** rustc-dev-guide (compiler/tooling), std-dev-guide (library).
- **Submodules/subtrees:** changes go to the component's own repo.
- **Bug reports / ICEs:** use the issue templates.

General flow: pick a `good first issue`/`E-mentor` issue → open a PR → `triagebot` assigns a reviewer and applies `S-waiting-on-*` labels → reviewer `r+` → merged via the bors/merge-queue "not rocket science" rule. **Joining a team** is by nomination from existing members after a track record of contributions — no application form.

## See also

- [[03-governance-and-business]] · [[34-release-management]] · [[07-glossary]]

Sources: blog.rust-lang.org/2023/06/20/introducing-leadership-council/ · rust-lang.github.io/rfcs/3392-leadership-council.html · www.rust-lang.org/governance · rustc-dev-guide.rust-lang.org · in-repo `CONTRIBUTING.md`, `triagebot.toml`
