# Implementation Spec — Reply Debt

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Owe** (working title; app id `replydebt`) · proprietary |
| Stack | Next.js 15 PWA + TS 5; local-first: Dexie 4 (IndexedDB) primary + sync to Postgres 16 (Drizzle) via REST batch endpoint; Auth.js magic link; worker for digests |
| Shared core | `packages/core` — scoring, drift, sync merge; imported by client + server (single source of truth) |
| Sync | field-level LWW with `updated_at` + tombstones; batch push/pull `since` cursor; client-generated UUIDv7 ids |
| Scoring constants | `urgency = age_days × tierWeight{inner:3, close:2, everyone:1} × boost` (boost: time_sensitive 2.0, they_asked_question flag 1.5, else 1.0); paydown list = top 5 |
| Drift | formal definition in §8; dashboard threshold ≥ 1.25; digest weaves in max 1 drifted person/day (highest drift, not nagged in last 7 d) |
| Deep links | `whatsapp://send?phone=`, `sms:`, `mailto:`, `tel:`, `instagram://user?username=` with `https://wa.me/` etc. web fallbacks; matrix in `core/deeplinks.ts` |
| OCR | `tesseract.js` client-side lazy-loaded; text stored with debt, never uploaded unless sync on (screenshots synced to S3 encrypted at rest, per-user key envelope) |
| Push | 1 daily digest (user hour, default 08:30) + same-day time-sensitive; hard cap 2/day enforced server-side |
| Streak-free stats | drift dashboard only; no counters that punish |
| Ports | web 3000, worker 3007 |

## 1. Repository layout

```
replydebt/
  src/
    app/
      (app)/{today/page.tsx, people/page.tsx, person/[id]/page.tsx, drift/page.tsx, settings/page.tsx}
      share/page.tsx                  # PWA share-target (POST)
      api/{sync/route.ts, push/…, export/route.ts, blob/[key]/route.ts}
    components/{DebtCard.tsx (deep-link btn, Paid/Snooze/Drop), CaptureSheet.tsx (person picker fuzzy),
                PersonHeader.tsx, NotesEditor.tsx (markdown), TimelineList.tsx, DriftRow.tsx,
                TierPicker.tsx, ChannelEditor.tsx, BeforeYouCall.tsx, VacationBanner.tsx}
    db/{dexie.ts (schema+hooks stamping updated_at), sync.ts (push/pull loop, 30 s + on-focus)}
    lib/{ocr.ts, search.ts (minisearch index over people/notes/debts/ocr)}
  packages/core/src/{scoring.ts, drift.ts, merge.ts, deeplinks.ts, types.ts}
  server/{schema.ts (mirror tables + user_id), worker/digest.ts}
  e2e/
```

## 2. Dependencies

`next, react, dexie, dexie-react-hooks, drizzle-orm, postgres, next-auth@5, zod, tesseract.js, minisearch, web-push, @aws-sdk/client-s3, nanoid/uuidv7 (uuid pkg), date-fns, tailwindcss, marked (notes render)`. Dev: vitest, fast-check, playwright.

## 3. Client schema (Dexie stores; server mirrors + `user_id`)

```ts
people: 'id, name, tier, archived, updated_at'
  // {id, name, avatarRef?, tier, cadenceDays?, channels: [{kind, handle}], notesMd, dates: [{label, date}],
  //  archived, updated_at, deleted}
debts: 'id, personId, status, createdAt, updated_at'
  // {id, personId, direction: 'i_owe'|'they_owe', note, screenshotRef?, ocrText?, sourceHint,
  //  timeSensitive, theyAskedQuestion, dueHint?, snoozedUntil?, status: 'open'|'paid'|'dropped',
  //  paidAt?, createdAt, updated_at, deleted}
interactions: 'id, personId, at, updated_at'   // {kind: 'replied'|'met'|'called'|'note', note}
meta: 'key'                                    // cursor, deviceId, settings cache
outbox: '++seq'                                // unsynced row refs
```

## 4. Sync contract

`POST /api/sync {deviceId, push: Row[], pullSince: cursor}` → `{applied: n, pull: Row[], cursor}`. Row = `{table, id, data|null (tombstone), updated_at}`. Merge (core/merge.ts) is a **total, commutative order** so any device applying the same set of rows converges identically — evaluate these keys in order (first difference wins):

```
function winner(a, b):                  # a = incoming, b = existing; returns keeper
  if a.updated_at != b.updated_at: return later(a, b)      # 1. newest wins
  if a.deleted != b.deleted:       return the deleted one   # 2. equal ts → tombstone wins
  if a.deviceId != b.deviceId:     return lexicographically-greater deviceId  # 3. stable tiebreak
  return a                          # 4. identical origin → idempotent, either is fine
```

Rationale: tie-break is checked *before* deviceId only when timestamps are exactly equal, so "tombstone wins" and "deviceId lexical" never contradict — they are ordered rungs, not competing rules. Two-device divergence suites (edit/edit, edit/delete, delete/edit) in core tests assert commutativity (apply order A-then-B == B-then-A) and the no-lost-debts invariant: a row acked by server is never silently dropped (pull replay test).

## 5. Key flows

**Capture:** share-target POST (title/text/image) or FAB → CaptureSheet: person picker (minisearch fuzzy, recent-first, inline create), optional note, flags (time-sensitive, they-asked); image → local blob + async OCR → update row. ≤ 3 taps; offline-first (row written to Dexie immediately, outbox syncs).
**Paydown (today):** top-5 by scoring + all time-sensitive due today; DebtCard: person, age chip, note/thumb, deep-link button (per person's primary channel; long-press = channel chooser), Paid (→ status + auto-interaction `replied`), Snooze (1d/3d/date), Drop (kind copy: "Let it go — closure counts."). 
**Digest push:** server worker at user hour: count open debts + top names + at most 1 drifted person → single push `{title: "3 debts today", body: names, deepLink: /today}`; skip if zero; vacation mode suppresses all.
**Person page:** PersonHeader (tier, cadence editor, channels), BeforeYouCall (pinned notes first = notes lines starting `📌` or `!pin`), TimelineList (debts+interactions merged), dates list.

## 6. Milestone task lists

**M0** — T1 scaffold + Dexie schema + core package + eslint boundary (core imports nothing from app); T2 auth + server mirror + sync endpoint + sync loop; T3 people CRUD + TierPicker + onboarding (pick-10-people flow using contacts picker API where available, manual fallback).
**M1** — T1 CaptureSheet + share-target + OCR pipeline + offline test; T2 scoring.ts + golden tests + today page + DebtCard actions; T3 deeplinks.ts matrix + per-platform manual test doc; T4 paid→interaction auto-log. *Core loop ships.*
**M2** — T1 push digests (server worker, cap enforcement tests, vacation mode); T2 snooze/drop flows + drop-reason optional tags; T3 drift.ts + drift dashboard + digest weaving rule (fake-clock tests); T4 archive person (instant de-nag test).
**M3** — T1 person page complete (notes markdown, dates, timeline, BeforeYouCall); T2 minisearch index (people/notes/debt notes/OCR; 500-people < 100 ms bench); T3 avatar/photo refs + S3 blob route (sync-on only).
**M4** — T1 export/import (JSON + screenshots zip; lossless round-trip test); T2 settings (digest hour, quiet, cadence defaults per tier); T3 iOS PWA verification pass (share-target + push; if share-target unsupported → document Shortcuts recipe + clipboard-paste capture path); T4 copy/tone review vs `docs/tone-checklist.md` (no red badges, no guilt language — greps for banned phrases in strings file). Tag v1.0.

## 7. Test mapping

core: golden scoring/drift + fast-check (age monotonicity, tier ordering). Sync divergence suites. Playwright mobile: share→capture→offline→sync→digest link→paid. Push cap release-blocking. Capture friction budget test: share-to-saved ≤ 3 interactions (scripted).

## 8. Drift algorithm (`core/drift.ts`)

Drift measures how overdue a relationship is relative to its intended contact rhythm. Pure function over a person + their interactions (no I/O), golden-tested.

```
function driftFor(person, now):
  cadence = person.cadenceDays ?? defaultCadence[person.tier]   # inner 7, close 21, everyone 60 (days)
  last    = maxInteractionDate(person) ?? person.createdAt      # any kind: replied/met/called/note
  if person.archived or person.tier == 'everyone' and no cadenceDays: return null  # opted out
  daysSince = floor((now - last) / 1 day)
  return daysSince / cadence                                    # 1.0 = exactly due, >1 = overdue
```

@TUNE(defaultCadence={inner:7, close:21, everyone:60}), @TUNE(driftThreshold=1.25). Dashboard lists people with `drift ≥ 1.25`, sorted desc. Digest weaving: pick the single highest-drift person **not** surfaced in a digest in the last 7 days (tracked via `interactions` of kind `note`/last-nudged marker in `meta`); ties broken by longer `daysSince`. `null` drift (opted-out / no cadence for `everyone`) never appears on dashboard or in digests. Fake-clock tests assert: overdue detection at the boundary, exclusion of nagged-in-7d, and that logging any interaction resets drift below threshold next tick.
