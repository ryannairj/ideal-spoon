# Depwatch — "is this dependency bump safe?" answered properly

**Category:** Dev tool · **Difficulty:** Medium · **Platform:** CLI + GitHub App, self-hostable

## 1. Vision

Renovate/Dependabot tell you a new version *exists*; they don't tell you whether upgrading will hurt. Depwatch answers the question you actually have: **what changed between my version and the target, does any of it touch APIs I use, and how risky is this bump?** It reads changelogs/release notes/commit diffs, cross-references *your* codebase's actual usage of the package, and produces a verdict: `safe / review points / breaking for you` — as a CLI report or a comment on the Renovate/Dependabot PR.

## 2. Why it can win

- The "read 14 changelogs" toil is universal, hated, and perfectly LLM-shaped — but the killer feature is grounding against **your usage**: "v5 removed `parseAsync` — you call it in 3 files (list)" beats a generic changelog summary.
- Composes with existing bots instead of replacing them: it's the review layer on top of Renovate — zero migration cost to adopt.

## 3. Users & use cases

Any team with a dependency-update queue; solo maintainers batch-upgrading quarterly.

1. `depwatch check zod@4` in a repo → terminal report: change summary, breaking changes, *your* affected call sites, verdict + suggested codemod notes.
2. GitHub App comments the same report on every Renovate/Dependabot PR automatically.
3. `depwatch audit` → rank all outdated deps by (risk × how far behind × how exposed your usage is) → a sane upgrade order.

## 4. MVP scope & non-goals

**MVP:** npm ecosystem end-to-end (CLI + GitHub App); evidence pipeline (changelogs, GitHub releases, commit log fallback, types diff); usage analysis (TS/JS via tree-sitter + optional `tsc` API-surface diff); risk verdicts with citations; PR comments; report cache shared per package-version-pair.
**Non-goals (MVP):** auto-fixing/codemods (describe, don't apply), ecosystems beyond npm (PyPI next — adapter interface from day 1), private registries (v1.1), vulnerability scanning (link `osv.dev` data, don't rebuild it).

## 5. Tech stack

- **Core:** TypeScript, Node 22; single package powering CLI (`commander`) and server (Fastify + Probot).
- **Evidence:** npm registry metadata; GitHub Releases/CHANGELOG.md/commits via API; `@definitelytyped`/bundled `.d.ts` diffing via `ts-morph` (exported-surface extraction).
- **Usage analysis:** tree-sitter to find import sites + member usage of the target package; confidence tiers (static import > require > dynamic).
- **LLM:** cheap model for changelog distillation; strong model only for the final "does this affect these call sites" reasoning; provider-agnostic; aggressive caching (per `pkg@from→to` evidence bundle is user-independent).
- **Store:** SQLite (CLI local cache) / Postgres (App).

## 6. Architecture & pipeline

```
target: pkg, from, to
1 evidence: releases/changelog entries covering (from, to]  → fallback: commit messages + PR titles
2 api-diff: exported .d.ts surface from vs to → removed/changed/renamed symbols (deterministic!)
3 distill: LLM → structured changes[{kind{breaking,feature,fix,deprecation}, summary, symbols[], evidence_url}]
   — validator: every `breaking` must cite evidence; api-diff removals are auto-injected as breaking even if changelog silent
4 usage: scan repo for imports of pkg → used symbols + call sites
5 verdict: join(changes.symbols × used symbols) → affected[{change, sites[{file,line,snippet}]}]
   verdict = breaking-for-you | review (behavioral/deprecations touching you) | safe (nothing you use affected)
6 render: terminal (rich) / markdown (PR comment) with citations
```

## 7. Data model

- `evidence_bundles(pkg, from_v, to_v, sources_json, changes_json, api_diff_json, distilled_at, model)` — the shareable cache.
- `checks(id, repo, pkg, from_v, to_v, usage_json, affected_json, verdict, created_at)`
- App: `installations`, `repo_settings(auto_comment, verdict_labels, ignore_pkgs)`.

## 8. Feature specs

**F1 — CLI check.** As §3.1; `--json` output for scripting; exit codes (0 safe, 1 review, 2 breaking-for-you) for CI gates. *AC:* `zod@3→4`, `express@4→5`, `eslint@8→9` fixture repos produce correct affected-site lists (golden tests); cold check < 60 s, cached < 5 s.

**F2 — API-surface diff.** Deterministic exported-symbol diff from typings; works when changelogs lie or don't exist. *AC:* removed export in fixture pkg is flagged breaking with zero LLM involvement; packages without types degrade gracefully to changelog-only with a stated confidence downgrade.

**F3 — Usage grounding.** Import styles: ESM named/default/namespace, CJS require, re-exports one level deep; per-site snippet + confidence. *AC:* precision spot-check on 3 real OSS repos ≥ 90% (a listed site really uses the symbol); dynamic-import-only usage reports "usage unknown — manual review" honestly.

**F4 — GitHub App.** Detect Renovate/Dependabot PRs (branch/actor patterns + lockfile diff parse) → comment report; updates comment on PR rebase; optional label + status check (`depwatch/verdict`, non-blocking by default). *AC:* lockfile diff parser handles npm/pnpm/yarn-berry fixtures; one comment per PR, edited not duplicated.

**F5 — Audit mode.** All outdated deps (vs `npm outdated` data) → checks (cached where possible) → ranked table + suggested batches (patch-safe batch, minors, each major solo). *AC:* ranking formula documented + unit-tested; 60-dep fixture audit completes < 10 min cold within a configurable token budget.

**F6 — Verdict quality bar.** Every breaking claim carries evidence links; "safe" never asserted when usage analysis had unknown-confidence sites (downgrades to review). *AC:* golden benchmark (20 labeled bumps incl. 5 with silent breaking changes) — breaking-for-you recall ≥ 0.9, safe-verdict false-positive rate 0 on the benchmark (asymmetric by design: wrong "safe" is the cardinal sin).

## 9. Milestones

- **M0 (1):** CLI skeleton, registry/evidence fetchers, changelog distillation with citations. *Ships: nice changelog summarizer.*
- **M1 (2):** api-surface diff + validator merge. 
- **M2 (3):** usage analysis + verdict join + golden fixtures. *Ships: the real product (CLI).* 
- **M3 (4):** GitHub App + PR comments + cache service.
- **M4 (5):** audit mode, JSON/CI mode, docs + self-host compose. *Ships: v1.0.* (M5: PyPI adapter.)

## 10. Testing

- Golden benchmark repos + labeled bumps as the F6 regression gate (recorded LLM fixtures in CI; nightly live).
- ts-morph surface extraction unit suite (overloads, namespaces, type-only exports).
- Lockfile-diff parsers fuzzed with real-world fixtures.

## 11. Risks & open questions

- Changelog quality varies wildly → the deterministic api-diff floor + commit-fallback keeps output non-empty; confidence labeling keeps it honest.
- Monorepos/workspaces (usage scan scope) → respect workspace globs; per-package usage attribution tested on a pnpm-workspace fixture.
- Token cost at audit scale → shared evidence cache is the economics: `pkg@from→to` distillation is computed once globally (hosted mode) or per-org (self-host).
