# Implementation Spec — Reddit Recall

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Rewind for Reddit** (working `redditrecall`) · License: proprietary; self-host compose shipped |
| Stack | Next.js 15 + TS 5; Postgres 16 + pgvector (Drizzle); Redis + BullMQ; Auth.js v5 (Reddit OAuth + email magic link) |
| Reddit access | user's own OAuth token (scopes `identity history read save`), refresh-token flow; token-bucket 55 req/min per user (headroom under 60), centralized limiter in Redis |
| Poll cadence | saved-list poll every 15 min per connected account (staggered) |
| Comment pruning (deterministic) | top-level sorted by score desc, take until 200 comments total, depth ≤ 3, min score 2; megathreads (>5k comments) take top 300 |
| Distill models | map (comment clusters): `deepseek-chat`; reduce (digest): `deepseek-chat`; embeddings `text-embedding-3-small` |
| Digest schema (zod) | `{tldr, consensus, top_answers:[{claim, support, caveats, permalink}], dissent:[…], action_items:[…]} ` — every top_answer/dissent must carry a permalink present in pruned set (validator) |
| Nudge policy | 3 d → 10 d → 30 d → auto-archive w/ digest mention; ≤1 push + ≤1 email per day per user |
| Email | Resend, weekly digest Sunday per-user-tz 8:00 |
| Collections | auto-suggest at cosine ≥ `@TUNE(autoCollectThreshold=0.78)` to centroid; centroids recomputed nightly + incrementally on accept (see §10) |
| Est read time | words/200 wpm on digest (not thread) |
| Ports | web 3000, worker 3004 |

## 1. Repository layout

```
redditrecall/
  src/
    app/
      (app)/{inbox/page.tsx, item/[id]/page.tsx, queue/page.tsx, collections/page.tsx,
             collections/[id]/page.tsx, search/page.tsx, settings/page.tsx}
      add/page.tsx  api/{auth/…, ingest/route.ts (paste/share/email-in webhook),
                         items/…, sse/route.ts}
    server/{routers or route handlers}/…   # route handlers +服务 modules:
    services/{redditClient.ts (OAuth, limiter), backfill.ts, poller.ts,
              fetchThread.ts, prune.ts, distill.ts, collections.ts,
              queueEngine.ts (nudges), digestComposer.ts, unfurl.ts (non-reddit)}
    components/{ItemCard, DigestView (tldr/answers/dissent sections), QueueRow,
                CollectionChips, NudgeSheet, ReadTimeChip, ConnectRedditCard}
  worker/src/{index.ts, jobs/{poll.ts, backfill.ts, distill.ts, nudge.ts, weekly.ts, centroids.ts}}
  packages/db/src/schema.ts
  fixtures/threads/*.json (10 diverse incl. deleted, NSFW-flag, megathread)
  fixtures/eval/rubric.md
  docker-compose.yml
```

## 2. Dependencies

`next, react, drizzle-orm, postgres, bullmq, ioredis, next-auth@5, zod, resend, web-push, snoowrap` NOT used (raw fetch client — snoowrap is stale; write `redditClient.ts` on fetch with typed endpoints), `open-graph-scraper` (non-reddit links), `@tanstack/react-query, tailwindcss, date-fns-tz`. Dev: vitest, playwright.

## 3. Configuration

`DATABASE_URL, REDIS_URL, REDDIT_CLIENT_ID/SECRET, AUTH_SECRET, RESEND_KEY, DEEPSEEK_API_KEY, OPENAI_API_KEY, VAPID_*, EMAILIN_WEBHOOK_SECRET`. User settings: digest day/hour, nudge policy on/off, quiet hours, auto-archive opt-out.

## 4. Database schema (concretions over PLAN §7)

```sql
CREATE TABLE users (id uuid PK, email text, reddit_username text, reddit_tokens jsonb,
  tz text DEFAULT 'UTC', settings jsonb DEFAULT '{}', created_at timestamptz);
CREATE TABLE items (id uuid PK, user_id uuid NOT NULL,
  source text CHECK (source IN ('reddit','web')), url text, reddit_fullname text,
  title text, subreddit text, author text, saved_at timestamptz, fetched_at timestamptz,
  status text DEFAULT 'pending' CHECK (status IN ('pending','fetched','distilled','failed','gone')),
  content jsonb, digest jsonb, read_state text DEFAULT 'unread'
    CHECK (read_state IN ('unread','reading','done','archived')),
  est_read_min int, embedding vector(1536),
  tsv tsvector GENERATED ALWAYS AS (to_tsvector('english',
    coalesce(title,'')||' '||coalesce(digest->>'tldr','')||' '||coalesce(content->>'flat_comments',''))) STORED,
  UNIQUE(user_id, reddit_fullname), created_at timestamptz DEFAULT now());
CREATE INDEX items_tsv ON items USING gin(tsv);
CREATE INDEX items_vec ON items USING hnsw (embedding vector_cosine_ops);
CREATE TABLE collections (id uuid PK, user_id uuid, name text, kind text CHECK (kind IN ('auto','manual')),
  centroid vector(1536));
CREATE TABLE item_collections (item_id uuid, collection_id uuid, source text CHECK (source IN ('auto','user')),
  PRIMARY KEY(item_id, collection_id));
CREATE TABLE nudges (id uuid PK, item_id uuid, due_at timestamptz, sent_at timestamptz,
  step int, action text);
CREATE TABLE digests (id uuid PK, user_id uuid, week date, item_ids uuid[], html text,
  sent_at timestamptz, opened_at timestamptz);
CREATE TABLE sync_state (user_id uuid PK, backfill_cursor text, backfill_done boolean DEFAULT false,
  last_poll_at timestamptz, newest_fullname text);
```

## 5. Route/API contract

Pages are server components reading Drizzle. Mutating routes: `POST /api/ingest {url}` (paste/share-target; also `POST` from email-in webhook with secret) · `PATCH /api/items/{id} {read_state|snooze_until}` · `POST /api/items/{id}/collections {collectionId|newName}` · `POST /api/collections/{id}/accept-suggestion` · `GET /api/search?q=` (hybrid) · `POST /api/push/subscribe`. SSE `/api/sse`: `item_ready {itemId}` for live status flips.

## 6. Pipelines (worker jobs)

**backfill.ts**: paginate `/user/{name}/saved` (100/page) via cursor in sync_state; resumable; enqueue fetch per item; rate-limited by shared bucket. **poller.ts**: newest-first until seen `newest_fullname`. **fetchThread.ts**: `GET {permalink}.json?limit=500&depth=3` → store post + raw comments → prune.ts (deterministic per §0, store pruned + `flat_comments` text) → deleted/removed detection (`[removed]` body or 404 → status `gone` if nothing cached). **distill.ts**: cluster top-level comment trees into ≤ 6 clusters (embed + kmeans-lite, pseudocoded in §10) → map summaries (prompt `fixtures/prompts/map.md`) → reduce digest (prompt `fixtures/prompts/reduce.md`, zod + permalink validator, 1 retry) → embed(title+tldr) → collection suggestion. **queueEngine/nudge.ts**: schedule per policy on `distilled`; cancel on read_state change; caps + quiet hours. **weekly.ts**: compose per §PLAN F5 (skip if zero activity); track opens via pixel.

## 7. Screens

Inbox (new+distilled, ItemCard: title, subreddit, TL;DR 2 lines, ReadTimeChip, collection chips) · Item (DigestView sections, each claim links out to comment permalink; footer: open on Reddit, Done/Archive/Snooze) · Queue (nudge-due sorted, quick-win ordering, swipe actions) · Collections (+detail) · Search · Settings (Reddit connect, digest/nudge prefs, export JSON). Empty-state onboarding: connect → backfill progress bar with count.

## 8. Milestone task lists

**M0** — T1 scaffold + schema + auth (email) + compose; T2 Reddit OAuth connect + token refresh + redditClient with limiter (unit-tested bucket); T3 paste-URL ingest → items(pending) list UI.
**M1** — T1 backfill (resumable, 1k-item test with recorded pages) + poller + dedupe; T2 fetchThread + prune (fixture snapshots incl. megathread + deleted); T3 status SSE + inbox states. *Ships saves mirror.*
**M2** — T1 cluster+map+reduce distill + validators + recorded fixtures; T2 DigestView + Item page; T3 eval protocol (`fixtures/eval/rubric.md`, 20 threads scored ≥4/5 human rubric — run and record results doc); T4 embeddings + gone-state handling.
**M3** — T1 read states + queue + nudge engine (fake-clock suites, caps) + push; T2 hybrid search (RRF over vec+FTS incl. comment text — distinctive-phrase test); T3 collections auto-suggest + centroids job + accept/rename/merge UI.
**M4** — T1 weekly digest composer + email templates (Gmail/Apple manual checklist doc) + opens tracking + unsub; T2 PWA share-target + email-in webhook + sender allowlist; T3 export; T4 Stripe single-plan stub behind `BILLING=off` flag; T5 self-host docs. Tag v1.0.

## 9. Test mapping

Recorded Reddit JSON fixtures drive fetch/prune/distill suites; nightly live eval on 3 public threads (drift watch). Rate-limit simulation (429 handling + backoff) release-blocking. Fake-clock nudge/digest suites. Playwright: connect(mock) → ingest → distilled → digest view → done.

## 10. Algorithms, prompt fixtures & eval rubric

**kmeans-lite (`distill.ts`)** — clusters the pruned top-level comment embeddings; deterministic given a fixed seed so recorded-fixture tests are stable:

```
function clusterComments(embeds):        # embeds = unit-normalized vectors, n = len
  if n <= 3: return [allInOneCluster]     # too few to cluster
  k = min(6, max(2, ceil(sqrt(n / 2))))   # cap 6 per §0; grows slowly with n
  centroids = kmeansPlusPlus(embeds, k, seed=1337)   # k-means++ seeding (not random)
  repeat up to 15 iterations:
    assign each point to nearest centroid by cosine distance
    recompute each centroid = mean of its points, re-normalize
    if max centroid shift < 1e-4: break   # convergence
  drop clusters with < 2 points (fold their points into nearest kept cluster)
  return clusters sorted by summed member score desc
```

`k` is chosen by the closed form above (no elbow search — keeps it cheap and deterministic); `@TUNE(kCap=6, minClusterSize=2)` calibrated against `fixtures/threads/` in M2.

**Centroid freshness (`collections.ts` / `centroids.ts`)** — to avoid stale nightly-only suggestions: on every `accept-suggestion` or manual add, incrementally update the centroid as a running mean (`c' = (c*m + v)/(m+1)`, re-normalized); the nightly job does the authoritative full recompute (mean of all members) to correct drift. A newly-distilled item is matched against **current** centroids immediately, so suggestions are never more than one accept stale.

**Auto-collect threshold** — `0.78` is a starting value (`@TUNE`); M3 T3 records precision/recall of suggestions against the fixture corpus and adjusts. Below threshold → no suggestion (never auto-assign silently).

**Prompt fixtures** (`fixtures/prompts/`, versioned `# v1`):
- `map.md` — per-cluster summary; vars `{{clusterComments}}`; output = `{summary, keyClaims:[{claim, permalink}]}`.
- `reduce.md` — digest synthesis; vars `{{postTitle}} {{clusterSummaries}}`; output = the Digest schema (§0); example shows every `top_answer`/`dissent` carrying a permalink drawn from input.

**Eval rubric (`fixtures/eval/rubric.md`)** — committed schema, 5 criteria scored 0/1 each (pass = ≥ 4/5): (1) TL;DR factually matches thread, (2) consensus reflects highest-scored answers, (3) every claim's permalink resolves to a real pruned comment, (4) dissent captured when present, (5) no fabricated info. Stored as YAML front-matter + prose per criterion so the eval harness can parse scores.

## 11. Error Recovery & Graceful Degradation

| Failure | Trigger | Backoff | Fallback | User-facing UX |
|---|---|---|---|---|
| Reddit API rate limit | HTTP 429 / bucket exhausted | honor `X-Ratelimit-Reset` / `Retry-After`; else base 2s ×2 cap 60s; shared Redis bucket pauses all users' jobs | requeue item, no data loss; backfill resumes from cursor | Connect card shows "syncing… (rate-limited, will resume)" |
| Reddit token expired | 401 | 1 refresh attempt via refresh-token | if refresh fails: mark account `needs reconnect`, pause polling | Settings shows "Reconnect Reddit" |
| Thread deleted/removed | `[removed]`/`[deleted]` body or 404 | no retry | `status='gone'`; keep cached digest if one exists, else show "no longer available" | Item card badge "removed on Reddit" |
| LLM distill (map/reduce) | non-JSON / zod-invalid / permalink validator fail / timeout 60s | 1 retry with error appended | `status='failed'`, item stays in inbox as raw pruned thread (readable, no digest) | Item shows "summary unavailable — view thread" |
| Embedding | 5xx/timeout | 2 retries base 2s ×2 | item saved without embedding; FTS-only search + no auto-collection; nightly backfill re-embeds | silent |
| Email/push send | Resend/web-push non-2xx | 3 retries base 5s ×2; web-push 410 → delete sub | drop notification (respect daily cap), log; digest still viewable in-app | none |
