# Implementation Spec — Voice Inbox

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Mumble** (app id `voiceinbox`) · proprietary; self-host compose shipped |
| Stack | Next.js 15 PWA + TS 5; Postgres 16 (Drizzle); Redis + BullMQ; S3 (audio); Auth.js magic link; grammY Telegram bot in worker process |
| STT | interface `Stt {transcribe(audio): {words: [{w, startMs, endMs}], text}}`; default hosted Whisper-class API (`STT_PROVIDER=openai_compat`, `whisper-1`-style); self-host: `faster-whisper` container (`STT_PROVIDER=local`, HTTP shim included in compose) |
| Audio | MediaRecorder opus/webm 32 kbps mono; chunked upload 1 MiB (resumable, outbox); max 15 min |
| Extraction model | `claude-sonnet-5` (accuracy over cost here — wrong dates are the cardinal sin); items zod schema per PLAN §5 with `span: [startWord, endWord]` |
| Date resolution | LLM outputs `due_raw` string ONLY; `chrono-node` resolves against `said_at` + user tz; ambiguous/failed → `due_at=null` + `needs_when=true` (UI "when?" chip); NEVER LLM-resolved dates |
| Item types | `task, event, note, shopping, reminder, tell` (fixed) |
| Auto-file guardrails | never auto-file: unresolved dates, `tell`/`reminder` types, confidence < rule τ (default 0.9); auto-filed → daily digest note |
| Destinations | `internal` (default), `ics` (events; per-user private feed `/api/ics/<token>.ics`), `webhook` (HMAC-SHA256 header `X-Mumble-Signature`, 5 retries expo backoff, dead → settings banner) |
| Telegram | voice/audio/text accepted; replies with per-item summary + inline "open inbox" button; chat linked via one-time code from settings |
| Retention | audio default 30 d (`keep transcript forever` on); "delete audio on confirm" mode; sweeper hourly |
| Ports | web 3000, worker 3010 |

## 1. Repository layout

```
voiceinbox/
  src/
    app/
      (app)/{record/page.tsx (default), inbox/page.tsx, lists/page.tsx, list/[kind]/page.tsx,
             capture/[id]/page.tsx (transcript + spans), search/page.tsx, settings/page.tsx}
      api/{captures/route.ts, captures/[id]/…, items/[id]/route.ts, rules/…,
           ics/[token]/route.ts, webhook-test/route.ts, telegram-link/route.ts, push/…}
    components/{RecordButton (waveform, pause), OutboxBadge, CaptureGroup, ItemCard
                (type chip cycler, title inline-edit, due chip / WhenChip, confirm/discard swipes),
                MergeHint, SpanPlayer (plays word-span slice), RuleSuggestion, DigestNote,
                ListView, IcsSetupCard, WebhookForm}
    services/{pipeline.ts (orchestration), sttClient.ts, extract.ts, dates.ts (chrono wrapper),
              route.ts (destinations), rules.ts, digest.ts}
    lib/{recorder.ts, outbox.ts, sw.ts}
  worker/src/{index.ts, jobs/{transcribe.ts, extractJob.ts, retention.ts, digestJob.ts},
              telegram.ts}
  packages/db/schema.ts
  fixtures/{rambles/ (40 audio + labels.yaml), llm/, dates.yaml (golden)}
  docker-compose.yml (+ faster-whisper profile)
```

## 2. Dependencies

`next, react, drizzle-orm, postgres, bullmq, ioredis, @aws-sdk/client-s3, zod, chrono-node, grammy, idb, web-push, ics (feed gen), date-fns-tz, minisearch NOT — Postgres FTS, tailwindcss, wavesurfer.js (waveform + span playback)`. Dev: vitest, fast-check, playwright.

## 3. Database schema (concretions)

PLAN §7 with: `captures.transcript jsonb {words:[{w,s,e}], text}`; `items.span int4range` (word indexes); `items.state` transitions service-guarded (`draft → confirmed|discarded|merged`; `auto_filed → confirmed(revert-to-inbox resets to draft)`); `internal_entries(id, user_id, kind, content, done, due_at, topic, item_id)`; `rules(id, user_id, match jsonb {type?, keyword?, minConfidence}, action jsonb {autoFile: bool, destination?, topic?}, hits int, enabled bool)`; `events_ics(uid, user_id, item_id, dtstart, dtend?, summary)`; `telegram_links(user_id, chat_id, linked_at)`; unique ics token per user in `users.ics_token`.

## 4. Pipeline contract

```
capture uploaded (or telegram file fetched) → transcribe.ts (STT iface) → captures.transcript
→ extractJob: prompt(transcript.text + word count) → items[] zod (retry 1 w/ errors)
  span sanity: 0 ≤ start < end ≤ wordCount; overlaps > 10% between items → keep higher-confidence, flag other
  → dates.ts per item: chrono(due_raw, {instant: said_at, timezone: user.tz}) →
     unambiguous single result → due_at; else needs_when
→ rules.ts: matching enabled rule ∧ guardrails pass → auto_file (route + digest note)
   else state=draft → inbox; push "3 items from your voice note" (respect quiet hours)
→ telegram origin: bot reply with summary lines + inbox button
confirm (single or all-per-capture) → route.ts: internal insert | ics upsert | webhook POST
merge hint: new item title trigram ≥ 0.55 vs open items same type within 7 d → MergeHint chip
```

## 5. API contract (key routes)

`POST /api/captures` (chunked: init → PUT chunks → complete `{saidAt, source}`) · `GET /api/captures/[id]` (transcript + items) · `PATCH /api/items/[id]` `{title?|type?|dueAt?|state?|who?}` (state guard) · `POST /api/items/[id]/confirm-all` (per capture) · `POST /api/items/[id]/revert` (auto-filed → draft) · rules CRUD + `POST /api/rules/suggest-accept` · `GET /api/ics/[token].ics` (RFC5545; validates in gcal/apple — fixture test with `node-ical`) · `POST /api/webhook-test` · `GET /api/search?q=` (FTS transcripts + items, hit → `{captureId, wordOffset}`) · telegram link: settings generates code → user sends `/link <code>` to bot.

## 6. Screens & UX contracts

Record (default page): giant RecordButton, live waveform (wavesurfer), elapsed, pause/resume, done → immediate "processing" card with skeleton; offline → queued badge. Inbox: CaptureGroups newest-first; ItemCard interactions: swipe right = confirm, left = discard (with optional why-tags feeding rule suggestions), tap type chip cycles types, tap due chip = date sheet, WhenChip opens "when?" quick options (today/tomorrow/next week/pick); "Confirm all" per group; clearing 5-item capture ≤ 6 taps (scripted E2E). Capture detail: full transcript with item spans highlighted; tap item → SpanPlayer plays that slice (±2 s tolerance test while audio retained). Lists: minimal internal lists (due-sorted tasks, checkable shopping, notes by topic, reminders, tell-list grouped by person). Settings: destinations per type, ICS setup card, webhook form + recipes links (Todoist/Things/Notion glue docs), retention, quiet hours, telegram link, rules manager (hits count, disable, revert-created-by note).

## 7. Milestone task lists

**M0** — T1 scaffold + auth + schema + compose (+whisper profile); T2 recorder + chunked upload + outbox (airplane test) + captures list; T3 STT interface + both providers + transcript view; T4 FTS search over transcripts. *Voice memos with search ships.*
**M1** — T1 extract prompt + zod + span sanity + recorded-fixture suite vs `labels.yaml` (segmentation F1 ≥ 0.85 gate); T2 inbox UI + ItemCard flows + confirm-all + ≤6-taps E2E; T3 internal lists + routing; T4 push notifications + quiet hours.
**M2** — T1 dates.ts + `fixtures/dates.yaml` golden (tz/DST, "arvo"-style colloquial set, ambiguity → needs_when) — release gate; T2 WhenChip flow; T3 SpanPlayer (word-offset → ms via transcript timestamps) + alignment property test (spans tile, ≤10% overlap); T4 telegram bot (link flow, voice/text ingestion, replies) + bot test harness.
**M3** — T1 ICS feed + events routing + calendar-validation fixture test; T2 webhook destination (HMAC, retries, dead-endpoint banner) + recipes docs; T3 per-type default destinations.
**M4** — T1 rules: suggestion engine (10× same confirmed pattern → RuleSuggestion), guardrail table tests (release-blocking), auto-file + DigestNote daily + one-tap revert(+disable); T2 retention sweeper (no-orphan-blob test) + delete-on-confirm mode; T3 merge hints; T4 self-host docs + record 40-ramble corpus (varied voices/noise, scripts in `fixtures/rambles/scripts.md`) → store via LFS. Tag v1.0.

## 8. Test mapping

Ramble corpus = F2 permanent gate (recorded STT+LLM fixtures in CI; nightly live drift report). Dates golden = cardinal-sin gate. Guardrails table + webhook HMAC + retention release-blocking. Playwright mobile: record → inbox → confirm → ICS/webhook fixture receipt; telegram flow via harness.
