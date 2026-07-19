# Implementation Spec — Token Furnace

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Token Furnace** (`furnace`) · License MIT |
| Stack | Node 22 + TS 5, pnpm; Fastify 5; BullMQ + Redis; SQLite via Drizzle (`better-sqlite3`); React+Vite dashboard embedded |
| Job types (MVP, fixed ids) | `docstrings`, `module-readmes`, `changelog-draft`, `todo-triage`, `test-gap-report`, `pr-digest` |
| Job module contract | each job = dir `src/jobs/<id>/` exporting `{plan(repoCtx, config): Unit[], prompts: {generate, verify}, assemble(units): Delivery, SPEC.md}` |
| Languages (analysis) | TS/JS, Python, Go via web-tree-sitter; dominant-language detection by LOC |
| LLM | default provider DeepSeek (`deepseek-chat`); verify = same model, different prompt; spot-check model `claude-sonnet-5` at rate `SPOT_CHECK_RATE=0.1`; all via provider registry with time-windowed pricing |
| GitHub | fine-grained PAT per repo (App optional later); PRs from branch `furnace/<job>/<YYYY-MM-DD>`; never touches default branch |
| Discount windows | per-provider `price_json.windows: [{days, fromUTC, toUTC, multiplier}]` |
| Budget | monthly cents per job; hard-stop mid-run → `partial` + resume |
| Redaction | pre-send regex pass (AWS keys, `sk-`, PEM, JWT-shaped, `password=`) replaces with `«redacted»`, logged count |
| Workspaces | `/var/lib/furnace/repos/<id>` persistent clones (pull, not re-clone) |
| Ports | server+UI 3200 |

## 1. Repository layout

```
furnace/
  src/
    server/{app.ts, routes/{repos.ts, providers.ts, jobs.ts, runs.ts, reports.ts}, auth.ts}
    scheduler/{cron.ts, gates.ts (discount+budget), enqueue.ts}
    runner/{run.ts (lifecycle+checkpoints), units.ts, llm.ts (map/verify/spot), redact.ts,
            deliver/{pr.ts, report.ts}}
    analysis/{repo.ts (clone/pull), treesitter.ts, coverage.ts (docstring scan),
              symbols.ts, gitfacts.ts (log/blame helpers)}
    jobs/
      docstrings/{index.ts, prompts.ts, SPEC.md}
      module-readmes/… changelog-draft/… todo-triage/… test-gap-report/… pr-digest/…
      custom/loader.ts               # v1 YAML jobs
    llmproviders/{registry.ts, openaiCompat.ts, pricing.ts}
    db/{schema.ts, migrations/}
  ui/src/pages/{Jobs.tsx, JobForm.tsx, RunDetail.tsx, Repos.tsx, Providers.tsx, Costs.tsx}
  fixtures/repos/{ts-underdoc.tgz, py-underdoc.tgz, go-underdoc.tgz}
  fixtures/llm/                       # recorded generations incl. seeded-bad set
  docker-compose.yml
```

## 2. Dependencies

`fastify, bullmq, ioredis, drizzle-orm, better-sqlite3, web-tree-sitter, simple-git, @octokit/rest, zod, yaml, tiktoken, croner` (cron parse/next), `react, @tanstack/react-query, recharts` (cost graph), `tailwindcss`. Dev: vitest, playwright.

## 3. Configuration

Env: `FURNACE_DATA_DIR, REDIS_URL, ENCRYPTION_KEY (provider keys, PATs), SPOT_CHECK_RATE, TZ=UTC`. Job config zod (per type extends base): base = `{paths: string[], exclude: string[], schedule_cron, discount_only: bool, budget_cents_month, delivery: 'pr'|'report', style_guide?: string}`.

## 4. Database schema

```sql
CREATE TABLE repos (id text PK, url text UNIQUE, default_branch text, auth_ref text,
  language_stats text, last_pulled_at int);
CREATE TABLE providers (id text PK, name text, base_url text, key_enc blob, model text,
  price_json text);  -- {inPerM, outPerM, windows:[{days:[0-6], fromUTC:"16:30", toUTC:"00:30", multiplier:0.5}]}
CREATE TABLE jobs (id text PK, repo_id text, type text, config_json text, schedule_cron text,
  discount_only int DEFAULT 0, budget_cents_month int, delivery text, provider_id text,
  enabled int DEFAULT 1, created_at int);
CREATE TABLE runs (id text PK, job_id text, started_at int, finished_at int,
  status text CHECK(status IN ('running','ok','partial','failed','skipped_budget','skipped_no_work')),
  units_total int, units_done int, tokens_in int, tokens_out int, cost_cents real,
  output_ref text, checkpoint_json text);
CREATE TABLE units (run_id text, key text, status text CHECK(status IN ('pending','done','failed','rejected')),
  tokens int, verify_verdict text, spot_verdict text, error text, PRIMARY KEY(run_id, key));
CREATE TABLE reports (id text PK, run_id text, title text, body_md text, created_at int);
CREATE TABLE budget_ledger (month text, job_id text, cost_cents real, PRIMARY KEY(month, job_id));
```

## 5. API contract (`/api`, session cookie; single admin user)

`POST /repos {url, pat}` + `POST /repos/{id}/test` · `POST /providers` + `/test` · `GET/POST/PUT /jobs`, `POST /jobs/{id}/run-now`, `POST /jobs/{id}/pause` · `GET /runs?jobId`, `GET /runs/{id}` (units, cost, log tail), `POST /runs/{id}/resume` · `GET /reports/{id}` · `GET /costs?month` (per provider/job) · `GET /healthz`. Kill switch: `POST /admin/pause-all`.

## 6. Run lifecycle (normative, `runner/run.ts`)

```
gate: cron fire → discount window? (if discount_only) → month budget remaining > 0 → else skipped_*
plan: pull repo → job.plan() → Unit[] {key, input payload}   // deterministic; empty → skipped_no_work
loop units (resume from checkpoint; batch parallel 4):
  redact(input) → generate (cheap) → verify prompt (checklist critic → pass|reject+reason)
  reject → 1 regenerate with reason → still reject → unit failed (never ships)
  pass → maybe spot-check (rate) → strong-model verdict recorded (metrics only, unless reject → unit failed)
  budget check per unit: projected cost > remaining → checkpoint + status partial, stop
assemble: job.assemble(done units) → Delivery {files: [{path, content}] | report_md}
deliver: pr.ts (branch, commit-per-module, PR body w/ before/after metrics, label 'furnace')
         or report row (+optional commit to reports/ dir if configured)
account: tokens×window-price → runs + budget_ledger
```

## 7. Job specs (planner + verifier essentials; full detail in each SPEC.md — write these files verbatim from here + expand)

- **docstrings**: plan = tree-sitter public symbols lacking doc comment (per-language matchers); unit = symbol + full body + 2 nearest documented examples (style anchors). Verify checklist: params/returns match signature; no invented behavior (verifier sees body); style matches anchors. Assemble: insert docstrings, one commit per module dir; PR body = coverage before/after table.
- **module-readmes**: plan = dirs ≥ 5 files; drift gate: hash(file list + exported symbol names) vs stored → unchanged dirs skipped. Generate from file summaries (map) → README.md per dir.
- **changelog-draft**: plan = commits base..head grouped by conventional-commit type + path area; unit = group. Output `CHANGELOG.draft.md` w/ PR links; verify: every bullet cites ≥1 commit sha.
- **todo-triage**: plan = grep TODO/FIXME/HACK + blame age; cluster by file area; report ranked (age × marker severity) with owners (last author).
- **test-gap-report**: plan = exported symbols minus symbols referenced in test files; report lists top-N by rank (reuse PageRank-lite on import graph) + suggested cases (generate) with "suggestion only" framing.
- **pr-digest**: plan = merged PRs in window; report grouped by area, risk callouts (size, files touched in hotspots).

## 8. Milestone task lists

**M0** — T1 scaffold + schema + admin auth; T2 repos (clone/pull, PAT enc) + providers + pricing/windows (`gates.ts` + fake-clock tests incl. month boundary + midnight-crossing window); T3 scheduler (croner) + run ledger + skipped paths; T4 CI + compose.
**M1** — T1 analysis: treesitter + coverage scan (3 langs); T2 docstrings job end-to-end (plan→generate→verify→assemble); T3 pr.ts delivery (branch naming, collision suffix `-2`, guard test: PR diff touches only planned files); T4 fixture repos + recorded-fixture regression (coverage 40→90%, zero hallucinated params via validator: every @param name ∈ signature); T5 verifier efficacy test (seeded-bad fixtures ≥95% rejected).
**M2** — T1 unit checkpointing + resume (kill-mid-run chaos test); T2 budget hard-stop mid-run → partial → resume next window E2E; T3 dashboard pages + cost graph + kill switch; T4 redact.ts + count surfacing.
**M3** — T1 changelog-draft + todo-triage + module-readmes (each: SPEC.md, golden fixtures, drift-gate test for readmes); T2 report delivery + reports UI.
**M4** — T1 test-gap + pr-digest; T2 custom YAML jobs (`plan: {glob, query?}, prompt, verify_prompt`; sandbox: no shell, template-only) + example ("translate comments to English") E2E; T3 spot-check auto-escalation (reject-rate > 20% in run → pause job + report); T4 docs + compose hardening. Tag v1.0.

## 9. Test mapping

Golden outputs per job on fixture repos (recorded LLM); nightly live quality run publishing metrics. Release-blocking: PR-scope guard, budget/resume E2E, verifier efficacy, window math.
