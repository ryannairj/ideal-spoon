# Implementation Spec — Flaky Detective

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Flaky Detective** (`flakyd` server, `flaky-upload` CLI/action) · License Apache-2.0 |
| Server | Go 1.23, chi, Postgres 16 (pgx + sqlc), embedded React dashboard; single binary + Docker; port 8686 |
| Uploader | static Go binary + composite GitHub Action `flaky-detective/upload@v1` (wraps binary download) |
| Formats | JUnit XML, Jest/Vitest JSON (`--json` reporter output), pytest junitxml (+ optional `pytest-flakyd` plugin later), `go test -json` |
| Test identity | `key = sha1(suite + '::' + name + '::' + params_norm)`; params_norm strips numerics/uuids/hex from bracketed params |
| Flake stats | Beta-Bernoulli posterior α,β with exponential decay (half-life 14 days, applied on update); flake_rate = α/(α+β) |
| State machine | `healthy → suspected (≥2 intra-commit disagreements in 14 d OR pass-on-retry ≥ 3) → confirmed (posterior flake_rate ≥ 0.02 with ≥ 30 obs) → quarantined → recovering (14 clean days) → healthy`; transitions need N clean days hysteresis (suspected→healthy: 7) |
| Damage | `flake_rate × blocked_runs_28d × avg_pipeline_wait_min` (wait from upload metadata duration or default 10) |
| GitHub | App for PR comments + quarantine PRs; comment only when ALL failures in run are `confirmed` |
| Quarantine strategies | jest/vitest: `flaky-quarantine.json` skip-list + provided reporter package `@flaky-detective/vitest-reporter`; pytest: `quarantine.txt` + tiny plugin `flaky-detective-pytest`; go: `flakyquarantine.txt` + `flakyskip.Check(t)` helper lib |
| Auth | project token (`fd_<32>`) for uploads; dashboard admin password + read-only share links |
| Retention | raw results 90 d; rollups forever |

## 1. Repository layout

```
flaky-detective/
  cmd/flakyd/main.go  cmd/flaky-upload/main.go
  internal/
    ingest/{api.go, junit.go, jestjson.go, gotestjson.go, normalize.go, idempotency.go}
    engine/{stats.go (posterior+decay), detect.go (signals), states.go (machine+hysteresis),
            damage.go, rename.go (similarity tracking)}
    ghapp/{webhook.go, prcomment.go, quarantinepr.go, strategies/{vitest.go, pytest.go, gotest.go}}
    store/{schema.sql, queries.sql (sqlc)}
    digest/{weekly.go, slack.go, email.go}
    simulation/                       # THE test-bed: generators + metric asserts
      {models.go (flake models), gen.go, eval_test.go}
  reporters/
    vitest-reporter/ (npm pkg)  pytest-plugin/ (pypi pkg)  goskip/ (go module)
  web/src/pages/{Overview.tsx, Flakes.tsx, TestDetail.tsx, Runs.tsx, Settings.tsx}
      components/{OutcomeSparkline.tsx, HistoryMatrix.tsx, DamageTable.tsx, ClusterList.tsx}
  action/{action.yml, entry.sh}
  fixtures/{junit/*.xml (mangled, huge, unicode), jest/*.json, gotest/*.jsonl}
```

## 2. Dependencies

Go: `chi, pgx/v5, sqlc (committed gen), rs/zerolog, google/go-github/v60, bradleyfalzon/ghinstallation`. Web: react, tanstack-query, tailwind, `uplot` sparklines. Reporters: minimal deps (vitest reporter: none beyond vitest API; pytest plugin: pytest only).

## 3. Configuration

Env: `DATABASE_URL, LISTEN=:8686, GITHUB_APP_ID/PRIVATE_KEY/WEBHOOK_SECRET (optional), ADMIN_PASSWORD, RETENTION_DAYS=90`. Per-project settings (dashboard): thresholds override, budget N, digest channel (slack webhook/email), quarantine strategy + repo path of skip-list file.

## 4. Database schema (key DDL)

```sql
CREATE TABLE projects (id uuid PK, repo text UNIQUE, token_hash text, settings jsonb DEFAULT '{}');
CREATE TABLE uploads (id uuid PK, project_id uuid, ci_run_id text, attempt int, job text,
  matrix_key text, commit_sha text, branch text, pr int, duration_min real, at timestamptz,
  UNIQUE(project_id, ci_run_id, attempt, job, matrix_key));
CREATE TABLE tests (id uuid PK, project_id uuid, key text, suite text, name text, file text,
  first_seen timestamptz, state text DEFAULT 'healthy', state_since timestamptz,
  muted boolean DEFAULT false, UNIQUE(project_id, key));
CREATE TABLE results (test_id uuid, upload_id uuid, outcome text CHECK (outcome IN ('pass','fail','skip','error')),
  duration_ms int, failure_msg_hash text, trace text, PRIMARY KEY(test_id, upload_id));
CREATE TABLE flake_stats (test_id uuid PK, posterior_a real DEFAULT 1, posterior_b real DEFAULT 1,
  last_decay_at timestamptz, runs_28d int, fails_28d int, retried_greens_28d int,
  damage real, updated_at timestamptz);
CREATE TABLE incidents (id uuid PK, test_id uuid, opened_at timestamptz, closed_at timestamptz,
  history jsonb, issue_url text, quarantine_pr text);
CREATE INDEX results_recent ON results(upload_id);
```

## 5. API contract

`POST /api/v1/results` (bearer project token; multipart or gzip body + query `format=junit|jest|gotest`, metadata headers `X-FD-Run, X-FD-Attempt, X-FD-Commit, X-FD-Branch, X-FD-PR, X-FD-Job, X-FD-Matrix, X-FD-Duration`) → `{accepted, tests, newFlakesDetected}` — idempotent per §4 unique. `GET /api/v1/flakes?state=&sort=damage` · `GET /api/v1/tests/{id}` (history matrix data, clusters by failure_msg_hash) · `POST /api/v1/tests/{id}/mute` · `GET /api/v1/status/{commit}` → for PR comment logic. Webhook: `check_suite.completed`/`workflow_run.completed` → evaluate PR-comment rule. `GET /api/v1/budget` → `{confirmedUnquarantined, budget, ok}` for the optional status check.

## 6. Engine contracts

**Signals (detect.go)** per new upload batch: (a) intra-commit disagreement: same test_id, same commit_sha, differing outcomes across uploads/attempts → strong (+3 pseudo-observations of flakiness); (b) pass-on-retry: fail in attempt n, pass in attempt n+1 same run → strong; (c) cross-commit decorrelation: fails on commits not touching test's file paths (needs changed-files metadata header `X-FD-Changed`, optional) → weak. Real-break protection: test failing consistently (≥ 3 consecutive uploads, 0 passes) post-commit X → NOT flaky; state untouched; excluded from suspected promotion (release-blocking cases in simulation).
**stats.go**: on each result: decay α,β by `0.5^(Δt/14d)` since last_decay, then α+=isFlakySignalWeighted, β+=isCleanPass. **rename.go**: new key with ≥ 0.8 trigram similarity to a disappeared key (same suite) → link candidate, surface in UI ("possibly renamed"), never auto-merge.

## 7. Milestone task lists

**M0** — T1 server scaffold + schema + token auth + JUnit ingestion (streaming parser, 50k results < 30 s bench) + idempotency; T2 uploader CLI + Action; T3 raw runs/results browser page; T4 fixtures corpus + malformed-rejection tests.
**M1** — T1 simulation test-bed FIRST (models: random p, time-of-day, order-dependent, infra-correlated; generator + labeled ground truth); T2 stats+detect+states to pass eval (`precision ≥ 0.9 recall ≥ 0.8 @ 30 runs`, real-break FP = 0) — iterate constants against sim, constants live in one `tuning.go`; T3 Flakes table + sparklines + damage sort.
**M2** — T1 remaining parsers (jest/gotest) + normalize/params_norm tests; T2 TestDetail (HistoryMatrix commit×run, failure clusters, incident timeline); T3 damage accounting (blocked_runs join) + Overview KPIs (flake-caused red %, retry minutes); T4 weekly digest (slack/email; numbers reconcile test).
**M3** — T1 GitHub App + PR triage comment (all-confirmed rule table tests; edits own comment; conservative: any unknown failure → silent); T2 status API + `flaky/budget` optional check (opt-in).
**M4** — T1 reporter packages ×3 (each repo has example project + E2E: quarantine entry → suite green while flake still fails → recovery un-quarantine PR after 14 clean days, fake-clock); T2 quarantine PR bot (adds entry + opens issue, links incident); T3 recovering-state scheduler; T4 docs + compose + demo seed script. Tag v1.0.

## 8. Test mapping

`internal/simulation/eval_test.go` runs in CI on every engine change (the crown jewel per PLAN). Parser fixtures incl. 50 MB junit stream. Fake-clock: decay, hysteresis, recovery. Comment-rule table tests release-blocking.
