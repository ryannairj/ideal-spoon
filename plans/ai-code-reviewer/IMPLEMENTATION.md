# Implementation Spec — AI Code Reviewer

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Product name | **Peregrine** (GitHub App slug `peregrine-review`) |
| License | AGPL-3.0 |
| Stack | Node 22 + TS 5, pnpm workspaces; Fastify 5 + Probot 13; BullMQ + Redis 7; Postgres 16 + Drizzle |
| Workspaces | `apps/server` (webhooks+queue), `apps/worker`, `apps/dashboard` (Next.js 15), `packages/{db,llm,analysis,pipeline,shared}` |
| Models | lenses: `deepseek-chat`; verify + thread Q&A: `claude-sonnet-5`; env-overridable via provider registry |
| Clone strategy | shallow (`--depth=50`) sparse checkout of changed paths + their directories, workspace `/tmp/peregrine/<reviewId>`, deleted after run |
| Comment budget | `max_comments` default 12; severity floor default `minor` |
| Config file | `.aireview.yml` at repo root, zod schema in `packages/shared` |
| Lenses (fixed set) | `correctness`, `security`, `performance`, `tests`, `api-contract` |
| Diff position mapping | own module `packages/analysis/src/diffmap.ts` mapping file+line → review position, exhaustive tests |
| Findings cap per lens | 15 pre-verify |
| Dashboard auth | GitHub OAuth, only users with installation access see repos |
| Ports | server 3100, dashboard 3101 |

## 1. Repository layout

```
peregrine/
  apps/server/src/{app.ts, webhooks.ts, queue.ts, github.ts (octokit factory)}
  apps/worker/src/
    index.ts
    review/{orchestrator.ts, contextBuilder.ts, lenses.ts, verify.ts,
            rank.ts, publish.ts, incremental.ts, threads.ts}
    learn/{feedback.ts, memory.ts, tuning.ts}
  apps/dashboard/src/app/{page.tsx, repos/[id]/page.tsx, repos/[id]/replay/page.tsx,
                          repos/[id]/muted/page.tsx, api/auth/…}
  packages/db/src/{schema.ts, migrations/}
  packages/llm/src/{provider.ts, prompts/{lens_*.ts, verify.ts, walkthrough.ts, thread.ts}}
  packages/analysis/src/{diffmap.ts, treesitter.ts, contextBundle.ts, lockfileDiff.ts}
  packages/pipeline/src/types.ts        # Finding, ReviewResult, ContextBundle (zod)
  packages/shared/src/{config.ts, constants.ts}
  bench/
    golden/                             # git bundles of benchmark repos + labeled PRs
    labels/*.yaml                       # {pr, findings:[{path,line,kind,must_find}], clean:bool}
    run.ts score.ts                     # precision/recall computation
  docker-compose.yml (pg, redis, server, worker, dashboard)
```

## 2. Dependencies

`probot`, `fastify`, `bullmq`, `ioredis`, `drizzle-orm`, `postgres`, `@octokit/rest`, `simple-git`, `web-tree-sitter` (grammars: ts/js/py/go/rust/java), `zod`, `yaml`, `tiktoken`, `p-limit`. Dashboard: `next`, `next-auth`. Dev: `vitest`, recorded-LLM fixture loader (own, in `packages/llm/testing.ts`).

## 3. Configuration

Env: `DATABASE_URL, REDIS_URL, GITHUB_APP_ID, GITHUB_PRIVATE_KEY, GITHUB_WEBHOOK_SECRET, DEEPSEEK_API_KEY, ANTHROPIC_API_KEY, TOKEN_BUDGET_PER_REVIEW=400000, MAX_DIFF_LINES=6000`. `.aireview.yml` schema:
```yaml
lenses: {correctness: on, security: on, performance: on, tests: on, api-contract: off}
ignore: ["**/*.gen.ts", "vendor/**"]
min_severity: minor          # info|minor|major|critical
max_comments: 12
language: en                 # comment language
tone: neutral                # neutral|friendly|terse
instructions: |              # repo conventions injected into prompts
  We intentionally ignore errors from Close().
```
Invalid config → PR comment with zod error + defaults used (event recorded).

## 4. Database schema (key tables, SQL-equivalent)

```sql
CREATE TABLE installations (id bigint PK, account text, settings jsonb DEFAULT '{}');
CREATE TABLE repos (id bigint PK, installation_id bigint, full_name text UNIQUE,
  config jsonb, config_error text, enabled boolean DEFAULT true);
CREATE TABLE reviews (id uuid PK, repo_id bigint, pr_number int, head_sha text,
  base_sha text, status text CHECK (status IN ('queued','running','published','failed','skipped')),
  trigger text, tokens_in bigint, tokens_out bigint, latency_ms int,
  github_review_id bigint, created_at timestamptz, published_at timestamptz,
  UNIQUE(repo_id, pr_number, head_sha));
CREATE TABLE findings (id uuid PK, review_id uuid, path text, start_line int, end_line int,
  lens text, severity text, title text, body text, suggestion_patch text,
  verify_confidence real, state text CHECK (state IN
    ('published','suppressed_verify','suppressed_cap','suppressed_muted','resolved','rejected')),
  github_comment_id bigint, fingerprint text);  -- fingerprint = hash(lens, normalized title, path)
CREATE TABLE feedback (id uuid PK, finding_id uuid, kind text CHECK (kind IN
  ('thumbs_up','thumbs_down','resolved_no_change','fixed')), source text, at timestamptz);
CREATE TABLE muted_patterns (repo_id bigint, fingerprint_prefix text, reason text,
  downvotes int, created_at timestamptz, PRIMARY KEY(repo_id, fingerprint_prefix));
CREATE TABLE repo_memory (id uuid PK, repo_id bigint, note text, source_finding uuid,
  embedding vector(1536), created_at timestamptz);
CREATE TABLE threads (finding_id uuid PK, comments jsonb);
```

## 5. Pipeline contract (worker)

```
on PR opened/synchronize (dedupe on X-GitHub-Delivery, skip if head already reviewed):
 1 fetch: PR meta, diff (files+hunks), linked issue body; skip if draft (configurable)
 2 clone shallow sparse at head
 3 contextBuilder: per changed hunk → ContextBundle {hunk, enclosingSymbol (tree-sitter),
   callers/callees (same-repo grep+symbol match, max 5 each), fileImports, prDescription}
 4 lenses (parallel, p-limit 3): prompt(lens, bundles chunked ≤ 12k tokens) → Finding[] (zod)
 5 verify: strong model per finding with FULL bundle → {verdict: confirm|reject|downgrade,
   confidence 0..1, rationale}; keep confirm ∧ confidence ≥ 0.7
 6 dedupe by fingerprint; drop muted_patterns matches (state suppressed_muted)
 7 rank: severity desc, confidence desc; apply min_severity + max_comments cap
 8 publish single GitHub review: summary body (TL;DR + walkthrough table) + inline comments
   (suggestion blocks only when patch applies cleanly to head — dry-run apply check)
 9 persist everything
on synchronize (incremental): diff base = last reviewed head; re-run steps 3–8 for changed
 hunks only; for prior published findings: if hunk region changed → re-verify still-present?
   fixed → reply "✅ appears resolved" + state resolved; still present → no repeat comment
on issue_comment / review_comment reply mentioning the bot or on its thread:
 threads.ts → answer grounded in stored bundle (re-clone only if needed), post reply
```

## 6. Dashboard screens

Repos list (enabled toggle, config status) · Repo detail: reviews table (status, findings, acceptance %, tokens, latency), charts (findings by lens/severity over time, acceptance trend) · Muted patterns (list + unmute) · Replay: pick any PR → dry-run (full pipeline, `status='skipped'`, nothing posted — guard: publish step hard-disabled by flag threaded through, tested) with side-by-side would-be comments.

## 7. Milestone task lists

**M0** — T1 Probot app + webhook → queue → worker echo; T2 clone+diff fetch + workspace lifecycle; T3 hello-world summary comment on fixture repo; T4 compose + CI + delivery-dedupe test.
**M1** — T1 diffmap.ts + exhaustive tests (new/renamed/deleted files, multi-hunk, edge positions); T2 tree-sitter contextBuilder; T3 correctness lens + zod outputs + recorded fixtures; T4 publish single review w/ inline comments + suggestion dry-run check. *Ships single-lens reviewer.*
**M2** — T1 verify stage + thresholds; T2 fingerprint dedupe + rank + caps; T3 config loader + error comment; T4 remaining 4 lenses (each: prompt + 5 recorded-fixture tests); T5 bench harness (`bench/run.ts` on golden bundles; CI gate: precision ≥ 0.7 @ recall ≥ 0.4 on `must_find` labels using recorded outputs; nightly live).
**M3** — T1 incremental reviews + resolution replies; T2 token metering proof-test (unchanged files not re-analyzed); T3 threads Q&A + 60 s latency budget (streaming post).
**M4** — T1 feedback capture (reaction webhooks + resolved-without-change detection on merge); T2 muted_patterns (3 downvotes same fingerprint-prefix → mute + dashboard surfacing); T3 repo_memory distillation job (weekly: downvoted-finding rationales → notes) + prompt injection; T4 dashboard + replay dry-run; T5 self-host docs + compose hardening. Tag v1.0.
**M5** — T1 bench expansion (20 labeled PRs / 3 languages); T2 per-repo threshold auto-tune (weekly job, bounded ±0.1); T3 GitLab adapter spike doc only.

## 8. Test mapping

`bench/` is the regression gate (PR-time recorded, nightly live with metrics in job summary). diffmap + publish idempotency (duplicate delivery → one review) are release-blocking suites. Dry-run guard test: replay pipeline with network-mocked octokit asserting zero write calls.
