# Reply Debt — a personal CRM for the messages you owe people

**Category:** Life improvement · **Difficulty:** Medium · **Platform:** Mobile-first PWA

## 1. Vision

Friendships don't end in fights; they end in unanswered messages. Reply Debt is a tiny personal CRM built around one honest metric: **who is waiting on you, and for how long**. It tracks the people you care about, the threads you've been meaning to answer, and the cadence you *want* per relationship ("talk to Mum weekly, Dan monthly") — then gives you a 5-minute daily "debt paydown" ritual instead of ambient guilt.

## 2. Why it can win

- Personal CRMs (Monica, Clay) are contact databases with birthdays; nobody centers the actual failure mode — **reply latency and drift**. Framing debt → paydown is the product.
- Deliberately message-platform-agnostic and manual-first: no scary inbox integrations required (privacy + feasibility), with a frictionless capture flow (share a screenshot/message → "I owe Sarah a reply") that takes 3 seconds.

## 3. Users & use cases

Busy people with scattered chats (WhatsApp/IG/SMS/email) and good intentions.

1. Get a message you can't answer now → share it to Reply Debt (share-target) or tap "+ I owe" → pick person → done. It nags *you*, not them.
2. Morning card: "3 debts: Sarah (6 days — about the wedding), Dad (2 days), Alex (today)". Tap → jump to the app you talk to them in (deep link) → mark paid.
3. Cadence drift: "You wanted monthly with Dan — it's been 7 weeks. Here's what you last talked about."
4. Before a call: person page shows notes ("kid's name: Theo; started new job in May").

## 4. MVP scope & non-goals

**MVP:** people (import from contacts picker or manual), debts (owe-them / they-owe-me optional), capture via share-target/quick-add with optional screenshot attachment + note, cadences per person, daily paydown view + push nudge, person pages with notes/interaction log, jump-out deep links (whatsapp/sms/mailto/tel/ig), streak-free gentle stats (drift dashboard), full export.
**Non-goals:** reading your inboxes automatically (no WhatsApp/Gmail scraping — capture is manual/share-based by design), sending messages for you, AI-drafted replies (v1.1 as optional assist, never auto-send), social graph, team/sales CRM features, birthdays-and-gifts feature creep (a single "dates" field only).

## 5. Tech stack

- **App:** Next.js 15 PWA + TypeScript; local-first: IndexedDB (Dexie) as primary store with background sync to server (Postgres via Drizzle) — the daily flow must work offline in a subway.
- **Auth:** Auth.js magic link. **Push:** Web Push. **OCR (optional):** screenshot attachments get client-side `tesseract-wasm` text extraction so debts are searchable — nothing leaves the device unless sync is on (screenshots synced encrypted-at-rest, flagged clearly).

## 6. Architecture

Thin: PWA (Dexie + sync engine w/ LWW per field + tombstones) ↔ REST sync endpoint ↔ Postgres. Notification worker computes daily digests server-side from synced state; all scoring logic is shared TS (`packages/core`) so client and server agree.

**Debt scoring (deterministic, explainable):** `urgency = age_days × person_weight × context_boost` — person_weight from tier (inner circle 3×, close 2×, everyone 1×), context_boost for debts flagged "they asked a question / time-sensitive". Paydown list = top-5 by urgency, capped to feel doable.

## 7. Data model

- `people(id, user_id, name, avatar_ref, tier{inner,close,everyone}, cadence_days NULLABLE, channels_json /*[{kind, handle, deeplink}]*/, notes_md, dates_json, archived)`
- `debts(id, user_id, person_id, direction{i_owe,they_owe}, note, screenshot_ref NULLABLE, ocr_text, source_hint{whatsapp,sms,email,irl,…}, created_at, due_hint NULLABLE, time_sensitive bool, status{open,paid,dropped}, paid_at)`
- `interactions(id, user_id, person_id, kind{replied,met,called,note}, note, at)` — feeds cadence math; "mark paid" auto-logs one.
- `nudge_state(user_id, last_digest_at, snooze_until, settings_json /*digest hour, quiet days*/)`

## 8. Feature specs

**F1 — Capture.** Share-target (text/image) → person picker (fuzzy, recent-first, or create) → optional note → saved in ≤ 3 taps; quick-add widget-ish home shortcut; screenshots OCR'd for search. *AC:* share-to-saved < 5 s; works offline (queued sync); OCR failure never blocks capture.

**F2 — Paydown view.** Daily list per scoring; each card: person, age, note/screenshot thumbnail, deep-link button ("Open WhatsApp"), actions: Paid / Snooze (1d/3d/pick) / Drop (with honesty prompt "let it go guilt-free"). *AC:* scoring golden tests; deep links open the right app on Android + iOS for whatsapp/sms/mailto/tel; "Paid" logs interaction + closes debt in one tap.

**F3 — Cadence & drift.** Per-person target; drift = days since last interaction ÷ cadence; drift dashboard sorted worst-first with last-topic context (latest note/debt); drifted people get woven into the daily digest (max 1/day to avoid overwhelm). *AC:* drift math fake-clock tested; archiving someone removes them from all nags instantly.

**F4 — Person pages.** Notes (markdown), channels with deep links, dates, timeline (debts + interactions), "before you call" summary block (pinned notes first). *AC:* full-text search across people/notes/debt-notes/OCR text returns in < 100 ms locally at 500 people.

**F5 — Nudges.** One daily digest push (configurable hour, skip if zero debts + zero drift), time-sensitive debts may fire a same-day reminder; hard cap 2 pushes/day; vacation mode. *AC:* cap enforced by tests; digest deep-links into paydown view.

**F6 — Data ownership.** Export everything (JSON + screenshots zip); import back; account delete = hard delete. *AC:* export→wipe→import lossless round-trip test.

## 9. Milestones

- **M0 (1):** scaffold, local-first store + sync engine, people CRUD + tiers.
- **M1 (2):** debts + capture (share-target, screenshots) + paydown view with scoring + deep links. *Ships: core loop.*
- **M2 (3):** push digests, snooze/drop flows, cadences + drift dashboard.
- **M3 (4):** person pages polish, search + OCR, dates. 
- **M4 (5):** export/import, vacation mode, onboarding (pick 10 people, set tiers in 60 s), iOS PWA hardening. *Ships: v1.0.*

## 10. Testing

- Shared-core scoring/drift: golden + property tests (monotonic in age; tier ordering preserved).
- Sync engine: two-device divergence suites (edit/edit, edit/delete) — no lost debts.
- Playwright mobile: share-target capture → offline → sync → digest deep link → paid.
- Deep-link matrix documented + manually verified per platform release.

## 11. Risks & open questions

- Manual capture discipline is the adoption risk → share-target friction must stay near-zero; measure captures/user/week as the north star; if it stalls, revisit read-only email integration as opt-in (v2 question).
- Guilt vs. help tone: copywriting reviewed for kindness (drops are celebrated as closure, drift never shames); no red badges with big numbers.
- iOS share-target/push PWA limits → verify M1; Capacitor wrapper is the fallback.
