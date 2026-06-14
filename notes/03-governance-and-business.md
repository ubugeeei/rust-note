# Who contributes & how Rust is funded

- **Upstream commit:** `4f8ea98e` (context only; this note is non-code)
- **Area:** project governance / business model
- **Date:** 2026-06-14

> Membership and org details change over time — treat specific company lists as
> "as commonly known", not a live roster.

## Question

Which organizations and people contribute to Rust, and what's the business
model behind it?

## Short answer

Rust is a community-run open-source project organized into teams, supported by
the non-profit **Rust Foundation** (founded Feb 2021). Contributors are a mix of
unpaid volunteers and engineers paid by companies that depend on Rust. There is
**no direct monetization of the language** — it's free, dual MIT/Apache-2.0 OSS;
money flows via Foundation membership dues/donations and a surrounding
commercial ecosystem (support, training, certified toolchains).

## Walkthrough

### Who builds it
- Community **team structure**: compiler, lang, libs, infra, types, etc. Since
  2023 a **Leadership Council** coordinates across teams (replacing the old core
  team).
- Contributors = **volunteers + company-paid engineers**. Historically led by
  Mozilla; today engineers from **AWS, Microsoft, Google, Huawei** and
  specialist firms like **Ferrous Systems, Embecosm** are paid to work on Rust.

### The Rust Foundation
- Independent non-profit (US 501(c)(6)), established **Feb 2021**.
- Stewards the **trademark**, funds **infrastructure**, hires security
  engineers, gives grants.
- **Founding members:** AWS, Google, Huawei, Microsoft, Mozilla. Many more join
  as dues-paying members over time (e.g. Meta, Arm, Toyota, ...).

### Business model
- The language/compiler is **free OSS**, licensed **MIT OR Apache-2.0**. Not
  sold, not directly monetized.
- Foundation income = **membership dues + donations**, spent on keeping the
  project running, not profit.
- Companies invest because **they depend on Rust** (strategic / cost reduction).
- Commercial ecosystem earns *around* Rust: e.g. **Ferrous Systems' Ferrocene**
  (an ISO 26262 / IEC 61508 *qualified* Rust toolchain for safety-critical &
  automotive), plus consulting, training, support, and vendor tooling.

In one line: **free OSS language, revenue via commercial services/support and
corporate sponsorship.**

## See also

- [[01-repo-layout]]
