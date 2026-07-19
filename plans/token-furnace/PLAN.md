# Token Furnace — put dirt-cheap LLM tokens to work while you sleep

**Category:** Dev tool / AI infra · **Difficulty:** Medium · **Platform:** Self-hosted server + CLI, web dashboard

## 1. Vision

DeepSeek-class tokens cost near-nothing (especially off-peak discount windows), yet sit unused. Token Furnace is a **background job engine for LLM busywork over your repos**: docstring coverage, module READMEs, changelog drafts, test-gap reports, code-comment audits, TODO triage, commit-message hygiene reports. You define recurring jobs; the furnace burns cheap tokens on schedule (optionally *only* during discount windows) and delivers output as PRs, reports, or files — every change reviewable, nothing auto-merged.

## 2. Why it can win

- Reframes cheap tokens from "worse chat" to **"free maintenance labor"**: quality bar for docs-drafts and reports is achievable by cheap models, especially with verify passes (also cheap).
- Nobody owns "scheduled LLM janitor jobs over a codebase" as a product. CI bots do one thing; this is a general engine with a job catalog.
- Cost-awareness is a feature: budget caps, off-peak scheduling, per-job token accounting — treats tokens like a utility bill.

## 3. Users & use cases

Solo devs and small teams with under-documented repos; OSS maintainers.

1. Nightly: "raise docstring coverage in `src/` by drafting missing docstrings" → morning PR with 20 docstrings, each traceable to its function.
2. Weekly: "regenerate `ARCHITECTURE.md` if the module graph changed" → PR only when drift detected.
3. On-demand: "summarize all TODO/FIXME comments into a triaged report."
4. Budget: ≤ $2/month, only during DeepSeek off-peak hours.

## 4. MVP scope & non-goals

**MVP:** server + worker, repo connections (git clone/pull by URL + token), job catalog (6 built-in job types below), scheduling with discount windows, budget caps, PR/report delivery via GitHub, dashboard, verify-pass pipeline.
**Non-goals:** auto-merge anything, arbitrary agentic coding (jobs are templated pipelines, not open-ended agents), computer-use (out — poor fit for unattended cheap models), non-GitHub forges in MVP (adapter interface only).

**Built-in job types (MVP):** `docstrings`, `module-readmes`, `changelog-draft` (from commit range), `todo-triage`, `test-gap-report` (untested public functions list + suggested cases), `pr-digest` (weekly summary of merged PRs → report).

## 5. Tech stack

- **Server/worker:** TypeScript, Node 22, Fastify + BullMQ + Redis, SQLite (self-host default) via Drizzle.
- **Analysis:** tree-sitter for symbol extraction & docstring coverage measurement (TS/JS, Python, Go); `git` CLI for repo ops; GitHub via REST (fine-grained PAT or GitHub App).
- **LLM:** provider registry (DeepSeek default, any OpenAI-compatible); pricing table incl. time-windowed discount pricing; generate-with-cheap → verify-with-cheap (different prompt) → optional spot-check-with-strong (sampled 10%).
- **Dashboard:** React + Vite served by server.

## 6. Architecture

```
scheduler (cron + discount-window gate + budget gate)
   → job run: plan (deterministic analysis: what needs doing)
   → chunk into task units (e.g. 1 function = 1 unit)
   → LLM map (generate) → LLM verify (checklist critic) → assemble
   → deliver: branch + PR (or report file / dashboard report)
   → account tokens & cost per unit
```

Key discipline: **deterministic pre-analysis decides the worklist** (e.g. exactly which functions lack docstrings); the LLM only fills templated slots. Everything delivered is diffable and cites its source symbol. Job runs are resumable (unit-level checkpoints).

## 7. Data model

- `repos(id, url, default_branch, auth_ref, language_stats_json, last_pulled_at)`
- `providers(id, name, base_url, key_enc, model, price_json /*incl. discount windows w/ tz*/)`
- `jobs(id, repo_id, type, config_json /*paths, thresholds, style guide*/, schedule_cron, discount_only bool, budget_cents_month, delivery{pr,report}, enabled)`
- `runs(id, job_id, started_at, finished_at, status{ok,partial,failed,skipped_budget,skipped_no_work}, units_total, units_done, tokens_in, tokens_out, cost_cents, output_ref /*pr url or report id*/)`
- `units(run_id, key /*symbol path*/, status, tokens, verify_verdict, error)`
- `reports(id, run_id, title, body_md)` · `budget_ledger(month, job_id, cost_cents)`

## 8. Feature specs

**F1 — Repo & provider setup.** Add repo (URL + PAT), add provider with price/discount windows; connection tests. *AC:* discount window "16:30–00:30 UTC" gates correctly across month boundaries (unit tests with fake clock).

**F2 — Docstrings job (flagship).** Coverage scan → units = undocumented public symbols → generate docstring in repo's dominant style (style inferred from existing examples, or config) → verify pass checks: describes actual params/returns, no hallucinated behavior (verifier sees the full function body) → PR with one commit per module, PR body lists coverage before/after. *AC:* on a fixture repo, coverage 40%→90%+; zero docstrings referencing nonexistent params (validator + tests); re-run with no changes → `skipped_no_work`.

**F3 — Scheduling, budget, resume.** Cron + discount gate + monthly budget cap (hard stop mid-run at cap → `partial`, resumable next window). *AC:* budget exhaustion mid-run resumes exactly at next window without redoing done units.

**F4 — Delivery.** PR mode: branch `furnace/<job>/<date>`, clean diff, labeled; report mode: markdown in dashboard + optional commit to `reports/` dir. Never pushes to default branch. *AC:* PR contains only intended files (guard test); force-push/branch-collision handled by suffixing.

**F5 — Remaining job types.** Each ships with: deterministic planner, prompt pack, verifier checklist, fixture-repo test. *(changelog-draft groups commits by area and links PRs; module-readmes regenerates only on structural drift — file list/symbol hash change; todo-triage extracts, ages via git blame, clusters, ranks; test-gap cross-references public symbols vs test files; pr-digest summarizes merged PRs with risk callouts.)* *AC per type:* documented in `jobs/<type>/SPEC.md` with golden outputs.

**F6 — Dashboard.** Jobs list w/ next-run + last-status, run detail (units, cost, output link), monthly cost graph per provider, kill switch. *AC:* cost figures reconcile with provider usage within 5%.

**F7 — Custom job templates (v1).** User-defined job = planner (glob + regex/tree-sitter query) + prompt template + verifier template, in a YAML file; sandboxed (no shell). *AC:* example custom job ("translate code comments to English") runs from YAML alone.

## 9. Milestones

- **M0 (1):** scaffold, repo/provider CRUD, scheduler with discount+budget gates, run ledger.
- **M1 (2):** docstrings job end-to-end incl. verify + PR delivery + fixture tests. *Ships: flagship demo.*
- **M2 (3):** resume/checkpointing, dashboard, cost accounting.
- **M3 (4):** changelog-draft, todo-triage, module-readmes. 
- **M4 (5):** test-gap, pr-digest, custom YAML jobs, docker-compose + docs. *Ships: v1.0.*

## 10. Testing

- Fixture repos (TS, Python, Go) with known gaps; golden-output snapshot tests using recorded LLM fixtures; nightly live runs publish quality metrics.
- Verifier efficacy test: seed deliberately-wrong generations → verifier must reject ≥ 95%.
- Clock/window/budget property tests; resume-after-kill chaos test.

## 11. Risks & open questions

- Cheap-model quality on niche languages → job config `min_quality_sample`: strong-model spot-check rate auto-raises if rejects spike; job pauses with a report instead of shipping slop.
- PR fatigue → drift-gated jobs (only PR when something changed) and weekly batching are defaults.
- Secrets in code sent to third-party API → per-repo redaction pass (regex for common secret shapes) + documented data-flow; self-host + local models supported via OpenAI-compatible endpoints.
