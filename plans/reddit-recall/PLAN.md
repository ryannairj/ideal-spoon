# Reddit Recall — your Reddit saves, actively digested instead of passively hoarded

**Category:** SaaS / life improvement · **Difficulty:** Medium · **Platform:** Web app + share-to extension/PWA

## 1. Vision

Reddit "save" is where links go to die: no search that works, no summaries, no reminder to ever return. Reddit Recall ingests everything you save (plus links you send it directly), fetches the post *and the comments* — where the real value lives — distills each into a digest ("what the thread concluded, best answers, dissent"), organizes by topic, and **actively schedules you to come back**: a weekly digest email and a read-queue with gentle spaced nudges.

## 2. Why it can win

- Read-later apps (Pocket-likes) store *articles*; none understand that a Reddit thread's value is the **comment consensus**. Comment-aware distillation is the wedge.
- Active loop vs passive pile: saved → summarized → queued → digested → reminded → marked done/kept. The product's KPI is items actually *processed*, not stored.

## 3. Users & use cases

Chronic Reddit savers: researchers-by-hobby (buy-it-for-life, homelab, travel, fitness advice threads).

1. Save a thread on Reddit (or paste/share a URL) → appears with TL;DR + top-answers digest within minutes.
2. Sunday email: "7 saves this week — 3-minute digest"; click through to full distillations.
3. Search "that thread about robot vacuums" → finds it by *content*, including comment content.
4. Collections auto-suggested ("Japan trip", "home network") with manual override.
5. Queue view: "you saved this 3 weeks ago and never opened it — read (4 min) or archive?"

## 4. MVP scope & non-goals

**MVP:** Reddit OAuth sync of saved items (poll) + manual add (paste URL, email-in address, PWA share target); fetch + distill pipeline; item view (digest + link out); collections (auto-suggest + manual); search; weekly digest email; queue with nudges; archive/done states.
**Non-goals:** posting/commenting to Reddit, full offline copies of media, other platforms (HN/Twitter adapters later — schema kept source-agnostic), social/sharing features, mobile native apps.

## 5. Tech stack

- **App:** Next.js 15 + TypeScript, Postgres + pgvector (Drizzle), BullMQ + Redis workers, Auth.js (Reddit OAuth + email).
- **Fetching:** Reddit OAuth API for the user's saved list & thread JSON (respect rate limits; the user's own token does the reading). oEmbed/OG fallback for non-Reddit links people paste.
- **LLM:** cheap model for distillation (map comments → reduce), embeddings for search/collections; provider-agnostic module.
- **Email:** Resend + MJML-ish templates.

## 6. Architecture & pipeline

```
sources: reddit poll (15 min) │ paste │ share-target │ email-in
  → items(status=fetched) → distill worker:
      thread JSON (post + top ~200 comments by score, depth ≤ 3)
      → map: comment-cluster summaries → reduce: digest{tldr, consensus, top_answers[{claim, support, caveats, comment_link}], dissent, action_items}
      → embed → auto-collection suggestion
  → queue engine: new items enter read-queue; nudge scheduler applies spaced policy (3d, 10d, 30d → then auto-archive with note)
  → weekly digest composer (email)
```

## 7. Data model

- `users(id, email, reddit_account, settings_json /*digest day, nudge policy, quiet*/)`
- `items(id, user_id, source{reddit,web}, url, reddit_fullname, title, subreddit, author, saved_at, fetched_at, status{pending,fetched,distilled,failed}, content_json /*post + pruned comments*/, digest_json, read_state{unread,reading,done,archived}, est_read_min, embedding)`
- `collections(id, user_id, name, kind{auto,manual}, embedding_centroid)` · `item_collections(item_id, collection_id, source{auto,user})`
- `nudges(id, item_id, due_at, sent_at, action{opened,archived,snoozed,ignored})`
- `digests(id, user_id, week, item_ids, html, sent_at, opened_at)`

## 8. Feature specs

**F1 — Ingest.** Reddit OAuth connect → backfill entire saved history (paginated, rate-limited, resumable) then 15-min polling; paste/share/email-in for arbitrary URLs. *AC:* 1,000-item backfill completes without rate-limit bans (token-bucket test); duplicates (same fullname/URL) merge, not double.

**F2 — Distillation.** As pipeline; digest must link each claim to its source comment (permalink). Comment pruning is deterministic (score/depth) so re-runs are stable. *AC:* eval set of 20 threads: human-rubric spot-check ≥ 4/5 on "captured the thread's actual consensus"; every top_answer carries a working permalink; deleted/removed threads produce a graceful "content gone — here's what we grabbed at save time" state when cached, else marked unavailable.

**F3 — Item & queue UX.** Item page: TL;DR → top answers → dissent → open-on-reddit; one-tap Done/Archive/Snooze. Queue sorted by (nudge due, then est_read_min ascending — quick wins first). *AC:* queue interactions are single-tap on mobile PWA; est_read_min within ±1 min of word-count heuristic.

**F4 — Collections & search.** Auto-suggest via embedding similarity to centroids (threshold), user can accept/rename/merge; hybrid search across titles, digests, and comment content. *AC:* search for a distinctive phrase that appears only in a comment finds the item; auto-collection precision spot-check ≥ 80% on fixture corpus.

**F5 — Weekly digest email.** Sections: new & distilled (grouped by collection), "still unread from before", one "rediscovery" (old archived gem). Plain-text-friendly, dark-mode safe. *AC:* renders in Gmail/Apple Mail (litmus-style manual checklist); unsubscribes/pauses honored immediately.

**F6 — Nudges.** Spaced policy per §6, per-item snooze, global quiet hours; auto-archive after final ignored nudge *with* a "rescued from archive" digest section so nothing silently vanishes. *AC:* policy transitions covered by fake-clock unit tests; nudge volume capped (≤ 1 push+? email/day).

## 9. Milestones

- **M0 (1):** scaffold, auth incl. Reddit OAuth, manual URL add, raw item list.
- **M1 (2):** saved-list backfill + polling; thread fetch + storage. *Ships: reliable saves mirror.*
- **M2 (3):** distillation pipeline + item page + eval harness. *Ships: the wow.*
- **M3 (4):** queue + nudges + read states; search + collections.
- **M4 (5):** weekly digest email, PWA share target + email-in, settings, billing stub (Stripe, single plan) if SaaS route. *Ships: v1.0.*

## 10. Testing

- Recorded Reddit JSON fixtures (10 diverse threads incl. deleted/NSFW-flagged/mega-threads) → pipeline snapshots; recorded-LLM fixtures in CI + nightly live eval.
- Rate-limit/backoff simulation tests; OAuth token refresh path.
- Fake-clock suites for nudges/digests; email snapshot tests.

## 11. Risks & open questions

- **Reddit API terms/pricing** is the existential risk: architecture keeps all Reddit reads on the *user's own OAuth token* (personal-use posture), caches content at save time, and the source adapter is swappable. Re-check ToS before any commercial launch; self-host mode is the fallback story.
- LLM cost per thread → prune hard (top comments only), cheap model, cache by thread revision; target < $0.005/item.
- Users may find summaries "good enough" and never click through — that's fine; the KPI is processed items, not clicks.
