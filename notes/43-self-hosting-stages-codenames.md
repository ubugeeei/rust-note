# Self-hosting stages & "codenames"

- **Upstream commit:** `4f8ea98e`
- **Area:** bootstrap stages; release naming
- **Date:** 2026-06-15

## Question

What are the "ranks" of self-hosting (the bootstrap stages), and does Rust use release codenames?

## Short answer

rustc is **self-hosted** — it's written in Rust and compiles itself — and the "ranks" people mean are the **bootstrap stages** `stage0` → `stage1` → `stage2` (and optionally `stage3`). And no: **Rust releases do not have codenames.** Releases are plain semantic versions (`1.x.0`); the only year-named things are **editions** (2015/2018/2021/2024), and the mascot **Ferris** (unofficial). Toolchains are identified by **channel** (nightly/beta/stable), not codenames.

## The self-hosting stages (the "ranks")

Because rustc is written in Rust, building it is staged (full detail in [[04-bootstrap]]):

| Stage | What it is | Built by |
|-------|-----------|----------|
| **stage0** | a prebuilt compiler **downloaded** from rust-lang.org (the previous `beta`) | — (fetched; see `src/stage0`) |
| **stage1** | compiler built from the current source | stage0 |
| **stage2** | the fully **self-hosted**, shippable compiler | stage1 |
| **stage3** (optional) | rebuild to verify it matches stage2 (reproducibility) | stage2 |

So the "rank" of a build is its stage number: higher stage = more steps removed from the borrowed stage0, and stage2 is the "real" compiler that ships. `./x.py build` defaults to stage1; `--stage 2` produces the self-hosted one. The bootstrap compiler info lives in `src/stage0` (`compiler_version=beta`, a date, and artifact hashes).

## Codenames: Rust doesn't use them

Unlike Ubuntu ("Jammy Jellyfish"), Android ("Tiramisu"), or macOS ("Sonoma"), **Rust releases have no codenames.** A release is just its version:

- **Versions**: `1.96.0`, `1.97.0`, … on a 6-week train ([[34-release-management]]); point releases `1.x.y`.
- **Channels** (not codenames): `nightly` / `beta` / `stable` — see `src/ci/channel`.
- **Editions** are the only year-named axis: **2015 / 2018 / 2021 / 2024** — opt-in compatibility epochs, not codenames ([[11-editions]]).
- **Ferris** 🦀 is the (unofficial) community mascot/crab; "Rustaceans" are community members. These are branding, not version codenames.

> If "コードネーム" was meant differently (e.g. internal tool names), note that the *tools* do have names — `bors` (merge bot), `x.py`/bootstrap (build system, see [[01-repo-layout]] for why "x"), `crater` (ecosystem regression testing), `triagebot`, `rustbot` — but these are tool names, not release codenames.

## See also

- [[04-bootstrap]] · [[34-release-management]] · [[11-editions]] · [[01-repo-layout]] · [[07-glossary]]
