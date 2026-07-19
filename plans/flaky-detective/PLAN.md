# Flaky Detective — find, rank, and quarantine flaky tests before they rot your CI

**Category:** Dev tool · **Difficulty:** Medium · **Platform:** Server + CI reporters, self-hostable

## 1. Vision

Every team has them: tests that fail 3% of the time, get retried into green, and slowly teach everyone to ignore CI. Flaky Detective ingests every test result from CI, statistically identifies flakes (same commit, different outcomes), ranks them by damage (failure rate × how often they block merges), auto-files quarantine PRs, and tracks whether "fixed" flakes actually stayed fixed.

## 2. Why it can win

- Big-tech has this internally (Google's flaky dashboards, Spotify's Flink pipelines); small teams have nothing but `retry: 3`. The self-hostable middle is empty.
- Damage-ranked, evidence-linked reports ("flaked 14× this week, wasted ~3.2 h of CI, last 5 traces attached") turn an invisible tax into a fixable backlog.

## 3. Users & use cases

Teams of 5–100 engineers with CI suites > 5 min.

1. CI uploads JUnit XML per run (one reporter line in the workflow) → dashboard fills itself.
2. Weekly report: top 10 flakes by damage, new flakes, fixed-and-stayed-fixed.
3. `quarantine` label suggestion → bot opens a PR adding the test to the skip/quarantine list with an issue link.
4. A dev checks "is this failure me or a known flake?" from the PR comment the bot leaves when a known flake fails their run.

## 4. MVP scope & non-goals

**MVP:** ingestion API + uploader CLI (JUnit XML, Jest/Vitest JSON, pytest, go test JSON), flake detection engine, dashboard, GitHub integration (PR comment "known flake, not you", quarantine PRs for supported frameworks), weekly digest, retry-cost accounting.
**Non-goals:** root-cause analysis via AI (v1.1 — traces + timing clustering hints first), running tests itself, non-GitHub forges initially, autofix.

## 5. Tech stack

- **Server:** Go 1.23 (chi), Postgres, single binary + Docker; embedded React dashboard.
- **Uploader:** tiny static Go binary / GitHub Action (`flaky-detective/upload@v1`) posting artifacts + metadata (commit, branch, run attempt, job, matrix key).
- **GitHub:** App for PR comments + quarantine PRs.

## 6. Detection engine (the core)

- Identity: test key = suite + name + (normalized) parameters; fuzzy rename tracking via similarity on failure history (flagged, not silent).
- **Flake signal:** for a given commit SHA (or same PR head), outcomes disagree across runs/attempts → strong signal. Secondary: failure de-correlated from code changes (fails on unrelated diffs, passes on re-run), pass-on-retry counts.
- Score: `flake_rate` (Beta-Bernoulli posterior with decay, so old behavior fades) and `damage = flake_rate × runs_blocked × avg_pipeline_wait`.
- States: `healthy → suspected → confirmed → quarantined → recovering → healthy`, with hysteresis (needs N clean days to progress) — all transitions evented and visible.

## 7. Data model

- `projects(id, repo, settings_json, token_hash)`
- `uploads(id, project_id, ci_run_id, attempt, commit, branch, pr, job, at, raw_ref)`
- `tests(id, project_id, suite, name, params_hash, file, first_seen, state, muted)`
- `results(test_id, upload_id, outcome{pass,fail,skip,error}, duration_ms, failure_msg_hash, trace_ref)`
- `flake_stats(test_id, window, runs, fails, retried_greens, posterior_a, posterior_b, damage_score, updated_at)`
- `incidents(id, test_id, opened_at, closed_at, state_history_json, issue_url, quarantine_pr_url)`

## 8. Feature specs

**F1 — Ingestion.** Formats: JUnit XML, Jest/Vitest JSON, pytest (`--junitxml` + native plugin), `go test -json`. Idempotent by (run_id, attempt, job). *AC:* 50k-result upload processes < 30 s; malformed files rejected with actionable errors; re-upload is a no-op.

**F2 — Detection.** As §6. *AC:* simulation suite — synthetic histories with known flake rates (1%, 5%, 20%) + real-failure regressions mixed in: detector achieves ≥ 0.9 precision / ≥ 0.8 recall distinguishing flakes from real breaks at 30 runs of history; genuine new breakage (fails deterministically post-commit X) is NOT flagged flaky (release-blocking cases).

**F3 — Dashboard.** Damage-ranked flake table (sparkline of recent outcomes, failure-message clusters, duration drift), test detail page (history matrix commit×run, traces, linked incidents), suite health overview (flake-caused red-run %, retry minutes burned, trends). *AC:* every number clicks through to the underlying runs.

**F4 — PR triage comment.** When a PR's failed tests are all `confirmed` flakes → single comment: "3 failures are known flakes (links) — likely not your change"; never comments when any failure is unknown. *AC:* comment logic covered by table tests; edits its own comment on subsequent runs.

**F5 — Quarantine automation.** Per-framework strategies: Jest/Vitest (skip-list file consumed by a provided reporter), pytest (marker file + plugin), Go (skip-list via provided TestMain helper). Bot PR adds the entry + creates an issue; `recovering` state (still runs, doesn't fail the build via reporter) tracks post-fix stability for 14 days before an un-quarantine PR. *AC:* end-to-end on fixture repos per framework: quarantine PR → suite green despite the flake still flaking → recovery flow re-enables.

**F6 — Weekly digest + budgets.** Email/Slack-webhook digest; optional "flake budget" check (`flaky/budget` status turns red if confirmed-unquarantined flakes > N) to force periodic cleanup. *AC:* digest numbers reconcile with dashboard; budget check never blocks by default (opt-in).

## 9. Milestones

- **M0 (1):** server + uploader + JUnit ingestion + raw results browser.
- **M1 (2):** detection engine + simulation test-bed + flake table. *Ships: visibility.*
- **M2 (3):** dashboard detail views, damage accounting, weekly digest.
- **M3 (4):** GitHub App: PR triage comments. *Ships: daily value in PRs.*
- **M4 (5):** quarantine automation (3 frameworks) + recovery loop + budgets + docs/compose. *Ships: v1.0.*

## 10. Testing

- The **simulation test-bed is the crown jewel**: generator produces synthetic result streams (configurable flake models: random, time-of-day, order-dependent, infra-correlated) → detector metrics asserted in CI on every engine change.
- Parser fixtures from real-world CI outputs (mangled XML, huge attachments, unicode test names).
- E2E: docker-compose + fixture repo GitHub Action run against a test server.

## 11. Risks & open questions

- Order-dependent/infra-correlated flakes look like real breaks → correlation features (agent, shard, time) in v1.1; MVP labels these `suspected` honestly rather than guessing.
- Quarantine mechanisms differ per framework forever → strategy interface + community recipes doc; only 3 first-party in MVP.
- Trust: one wrong "not your change" comment burns credibility → F4's conservative "all failures known-flaky" rule is deliberate.
