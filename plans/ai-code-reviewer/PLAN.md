# AI Code Reviewer — self-hostable CodeRabbit-style PR review bot

**Category:** Dev tool / clone · **Difficulty:** Hard · **Platform:** Server (GitHub App), self-hostable

> Scope note: the original idea said "code rabbit, wiz". Those are different products — AI PR review vs. cloud security posture. This plan is the CodeRabbit-style reviewer; a Wiz-lite would be its own (much bigger) plan. A light security pass is included as one reviewer lens (F4).

## 1. Vision

Install a GitHub App on your org; every PR gets a fast, structured first review: a summary, a walkthrough, and inline comments that find real bugs — with your own model keys, your own server, your code never leaving your infra.

## 2. Why it can win

- CodeRabbit is SaaS-only for most tiers; regulated teams want **self-hosted + bring-your-own-model** (Claude, DeepSeek for cost, local models).
- Quality wedge: most bots comment noise. We optimize for **precision over recall** — a two-stage generate→verify pipeline and a per-repo learning loop from 👍/👎 reactions and "resolved without action" signals.

## 3. Users & use cases

Teams of 3–50 engineers; OSS maintainers drowning in drive-by PRs.

1. PR opened → within ~2 min: summary comment + walkthrough table + inline findings.
2. Reviewer replies to a bot comment with a question → bot answers in-thread with context.
3. Push a fix commit → bot re-reviews incrementally, resolves addressed comments.
4. `.aireview.yml` tunes lenses, paths to ignore, severity threshold, tone.

## 4. MVP scope & non-goals

**MVP:** GitHub App (webhooks: PR opened/synchronize/comment), review pipeline, summary + inline comments via the Reviews API, incremental re-review, per-repo config, dashboard with metrics (findings, acceptance rate), Docker deploy.
**Non-goals:** GitLab/Bitbucket (adapter interface from day 1, implementations later), auto-fix commits (suggestion blocks only), full-repo audits, IDE integration, CI-gate blocking mode (report-only in MVP).

## 5. Tech stack

- **Server:** TypeScript, Node 22, Fastify + Probot (GitHub App plumbing), BullMQ + Redis queue, Postgres (Drizzle).
- **Analysis:** `git` shallow clones in ephemeral workspaces; tree-sitter for symbol/context extraction; LLM via provider-agnostic `llm/` module (strong model for verify, cheap for bulk).
- **Dashboard:** Next.js, GitHub OAuth.

## 6. Architecture & review pipeline

```
GitHub webhook → queue → worker:
  1 fetch PR meta + diff, clone at head (shallow, sparse)
  2 context build: for each hunk → enclosing symbols, callers/callees (tree-sitter), related files, PR description, linked issue
  3 lenses (parallel LLM passes over chunked diff):
     correctness · security · performance · tests · api-contract
  4 verify stage: strong model re-examines each candidate finding w/ full context → keep only "confident, actionable"; dedupe/merge
  5 rank + threshold (config), cap comments/PR (default 12)
  6 publish: one review (summary + walkthrough + inline comments w/ suggestion blocks where safe)
  7 record everything for the learning loop
```

- Incremental: on `synchronize`, only re-analyze changed-since-last-review hunks; check prior findings — resolved ones get a ✅ reply, still-present ones are not repeated.
- Thread replies: comment webhook → conversational answer grounded in that hunk's context bundle.

## 7. Data model

- `installations(id, account, plan, settings_json)` · `repos(id, installation_id, full_name, config_json, enabled)`
- `reviews(id, repo_id, pr_number, head_sha, status, cost_tokens, latency_ms, published_at)`
- `findings(id, review_id, path, start_line, end_line, lens, severity{info,minor,major,critical}, title, body, suggestion_patch, verify_confidence, state{published,suppressed,resolved,rejected})`
- `feedback(finding_id, kind{thumbs_up,thumbs_down,resolved_no_change,fixed}, source, at)`
- `threads(finding_id, comments_json)` · `repo_memory(repo_id, note, source_finding_id, embedding)` — distilled per-repo conventions ("this repo intentionally ignores err on Close()").

## 8. Feature specs

**F1 — Review publish.** Single GitHub review containing: TL;DR summary, changed-files walkthrough table (file → purpose of change), inline comments. *AC:* P50 open→published < 3 min for a 400-line diff; comments land on correct lines (diff-position mapping unit-tested against fixture diffs incl. renames).

**F2 — Precision pipeline.** Two-stage verify; suppressed findings stored with reasons. *AC:* on the golden benchmark (see §10) precision ≥ 0.7 at recall ≥ 0.4 for seeded bugs; noise cap respected.

**F3 — Incremental re-review + resolution tracking.** *AC:* pushing a commit that fixes finding X produces a "resolved" reply on X's thread and no duplicate finding; unchanged files are not re-analyzed (token metering proves it).

**F4 — Lenses & config.** `.aireview.yml`: enable/disable lenses, `ignore: [paths]`, `min_severity`, `max_comments`, `language` tone, custom instructions (repo conventions). Security lens covers OWASP-top-10-style diff issues (injection, authz gaps, secrets in code). *AC:* config change takes effect next review; invalid config → PR comment explaining the error, fallback to defaults.

**F5 — Learning loop.** 👍/👎 on comments + resolved-without-change tracked per lens/rule; repo_memory notes injected into future prompts; weekly auto-tuning of per-repo severity threshold. *AC:* a finding pattern downvoted 3× in a repo is suppressed there (visible in dashboard as "muted patterns").

**F6 — Dashboard.** Per repo: reviews, findings by lens/severity, acceptance rate, token cost, muted patterns management, replay a review (dry-run against any PR without publishing). *AC:* dry-run mode never posts to GitHub (guarded by test).

**F7 — Q&A in threads.** Replies grounded in hunk context; refuses speculation beyond the diff+context bundle. *AC:* reply latency < 60 s; answers cite file:line.

## 9. Milestones

- **M0 (1):** Probot app, webhook→queue→worker skeleton, clone+diff fetch, hello-world summary comment on a test repo.
- **M1 (2):** context builder (tree-sitter enclosing symbols), correctness lens, inline publishing with correct positions. *Ships: single-lens reviewer.*
- **M2 (3):** verify stage, ranking/caps, config file, remaining lenses. *Ships: MVP quality bar.*
- **M3 (4):** incremental re-review, resolution tracking, thread Q&A.
- **M4 (5):** feedback capture, repo_memory, dashboard + dry-run replay, Docker compose + self-host docs. *Ships: v1.0.*
- **M5 (6):** golden-benchmark expansion, auto-tuning, GitLab adapter spike.

## 10. Testing

- **Golden benchmark repo(s):** curated PRs with seeded, labeled bugs (off-by-one, race, injection, broken null-handling…) + clean PRs. CI computes precision/recall vs labels on every prompt/pipeline change — this is the regression gate.
- Recorded-LLM fixtures for deterministic unit tests; nightly live benchmark workflow posts metrics.
- Diff-position mapper: exhaustive tests (new files, renames, deletes, multi-hunk, suggestions at hunk edges).
- Webhook idempotency: duplicate deliveries must not double-post (dedupe on delivery ID).

## 11. Risks & open questions

- **Noise kills adoption faster than misses** — hence verify stage, caps, learning loop; default config conservative.
- Monorepo/huge PRs → hard token budget; over-budget PRs get summary + "top files only" with a notice.
- GitHub rate limits at org scale → per-installation queues with backoff; batch review API (one review, many comments) instead of N comment calls.
