# TabSweep — rehab for tab hoarders

**Category:** Life improvement / browser extension · **Difficulty:** Easy–Medium · **Platform:** Chrome/Edge/Firefox extension (MV3)

## 1. Vision

You have 340 tabs open. They're not tabs — they're guilt with favicons: things you meant to read, buy, reply to, or decide about. TabSweep turns the pile into a system: tabs age visibly, stale ones sweep themselves into a searchable, categorized shelf (closing the tab, keeping the intent), and a weekly 5-minute triage digest keeps the shelf honest. The browser gets fast again; nothing is ever truly lost.

## 2. Why it can win

- OneTab (dump list, no intelligence) and Tab Wrangler (auto-close, no shelf story) each have half the loop. The whole loop — **decay → sweep → categorize → digest → resurface or expire** — is the product. LLM-era addition: swept tabs get auto-categorized by *intent* (read / buy / reference / reply / watch), which is what makes the shelf navigable at 500 items.
- Zero-account local-first extension = trivially adoptable; optional sync later.

## 3. Users & use cases

Everyone with >30 tabs; researchers, ADHD-adjacent workflows, serial shoppers.

1. Install → amnesty flow: "sweep 297 tabs older than 2 weeks to the shelf now?" → browser breathes.
2. Tabs untouched for N days get a subtle aging indicator → at threshold, swept (whitelist: pinned, audible, whitelisted domains/windows never swept).
3. Shelf: grouped by intent + topic clusters ("7 mechanical keyboard tabs — still deciding?"), instant search, one-click restore (single or group).
4. Friday digest: "This week you swept 43 tabs. 5 look important (a form you half-filled, 2 docs from work domains). Keep / let go?"
5. Items untouched on the shelf for 60 days → "let go" pile → auto-expire after a grace notice (configurable; nothing deletes silently).

## 4. MVP scope & non-goals

**MVP:** MV3 extension (Chrome first, Firefox parity in M4): tab age tracking, sweep rules + whitelists, amnesty onboarding, shelf (local IndexedDB) with search/restore/expiry, intent categorization + topic clustering (heuristics first-class: domain packs + title patterns; optional BYO-key LLM enhancement), weekly digest page + badge nudge, import from OneTab export, full export (JSON/HTML bookmarks).
**Non-goals:** account/cloud sync in MVP (export/import covers migration; sync = v1.1 with E2E), mobile browsers, tab *grouping/workspace management* for active tabs (we manage the leaving, not the living — scope fence), reading-mode/article capture (store URL+title+favicon+scroll hint only), telemetry of any kind.

## 5. Tech stack

- **Extension:** TypeScript + WXT (MV3 tooling, multi-browser builds); React for shelf/digest/options pages; IndexedDB (Dexie) storage; `chrome.alarms` for schedules (MV3 service-worker-safe), `chrome.tabs`/`sessions` APIs.
- **Categorization:** layered — 1) heuristics: domain packs (shopping/video/docs/social/news, versioned JSON) + URL/title patterns (e.g. `/cart|checkout/` → buy) + embeddings-free keyword clustering for topics; 2) optional LLM (user-supplied OpenAI-compatible key incl. local Ollama URL): batch-categorize titles/URLs only (never page content), clearly labeled.

## 6. Architecture

```
service worker: tab event listeners (created/activated/updated) → last_active ledger (alarm-driven flush)
             alarms: decay evaluator (hourly) → sweep queue → notification-or-silent per settings
shelf pages (React) ← Dexie ← sweep records
digest generator (weekly alarm) → digest page + badge
```

MV3 discipline: service worker is stateless between wakes — all state in storage; alarm-driven, event-sourced `tab_ledger` so a killed worker never loses ages.

## 7. Data model (IndexedDB)

- `tab_ledger(tab_key /*windowId:tabId + url-hash fallback*/, url, title, favicon, first_seen, last_active, pinned, audible, window_label)`
- `shelf(id, url, title, favicon, swept_at, source{auto,manual,amnesty,import}, intent{read,buy,reference,reply,watch,unknown}, topic_cluster, note, state{shelved,kept,restored,let_go,expired}, expire_at, last_touched)`
- `rules(id, kind{whitelist_domain,whitelist_window,ttl_override,never_expire}, pattern, value)`
- `digests(id, week, stats_json, highlights_json, reviewed_at)` · `settings(...)` — TTL default 14 d, sweep style (silent/confirm), expiry 60 d + 7 d grace.

## 8. Feature specs

**F1 — Age tracking & decay.** Accurate last-active per tab across restarts (session restore remaps tab ids — reconcile by URL+index heuristics); optional subtle favicon-badge aging cue (off by default; MV3 limits → title-prefix fallback). *AC:* ledger survives browser restart & extension update (migration test); active tab never accrues staleness while focused.

**F2 — Sweep.** Threshold sweep honoring whitelists/pinned/audible/form-activity heuristic (tabs with `beforeunload` or detected input → require confirm); modes: silent (default after amnesty) or daily confirm batch; every sweep restorable for 30 s via toast and forever via shelf. *AC:* protected classes never auto-swept (table tests per rule); sweep of 200 tabs completes < 5 s without freezing the browser; restore reopens with original URL (and scroll position where `sessions` API allows).

**F3 — Amnesty onboarding.** First-run: scan → stats spectacle ("340 tabs, oldest 8 months") → one-click bulk sweep with age slider → whitelist quick-picks (work domains detected by frequency). *AC:* end-to-end in < 90 s; nothing closed until the single explicit confirm.

**F4 — Shelf.** Virtualized list; group by intent | topic | week | domain; instant fuzzy search; actions: restore, restore-group-to-new-window, keep (resets expiry), note, let-go; bulk select. *AC:* 2k items search < 50 ms; restored items marked (state machine tested); expiry grace notice lists exactly what will vanish, and "never_expire" rule wins over everything.

**F5 — Categorization.** Heuristic layer always-on; LLM batch pass (if key configured) runs on sweep batches, urls+titles only, results cached; user corrections are sticky (per-domain intent overrides learned locally). *AC:* heuristic layer alone ≥ 75% intent accuracy on the labeled fixture set (500 real-ish tabs); with LLM ≥ 90%; corrections outrank both (test).

**F6 — Weekly digest.** Page + badge count: stats, highlights (heuristics: form-activity tabs, work-domain docs, tabs you re-opened after sweeping = "you clearly need this"), the let-go pile preview, one-click triage rows. *AC:* digest builds from local data only; triage actions apply instantly; digest is skippable forever without breaking anything.

**F7 — Import/export.** OneTab export parser; bookmarks-HTML + JSON export of shelf; uninstall-safety doc (export nag before letting users leave, politely). *AC:* OneTab fixture (1k links) imports with dates preserved-ish (batch dated as import day, noted); export→fresh-install→import lossless.

## 9. Milestones

- **M0 (1):** WXT scaffold, ledger + age tracking, manual sweep to a basic shelf. 
- **M1 (2):** decay rules + whitelists + amnesty flow + toasts. *Ships: the rehab moment.*
- **M2 (3):** shelf UX (groups, search, expiry state machine) + heuristics categorization + fixture set.
- **M3 (4):** digest + LLM-optional pass + corrections; import/export. *Ships: MVP Chrome.*
- **M4 (5):** Firefox build + polish + store listings (screenshots, privacy manifest: "no data leaves your machine, period"). *Ships: v1.0.*

## 10. Testing

- Playwright + real Chromium with extension loaded: ledger/restart, sweep flows, restore, MV3 worker-kill resilience (force-terminate service worker mid-cycle).
- Labeled tab fixture set for categorization metrics (kept in-repo, versioned).
- State-machine table tests (shelf item lifecycle); alarm/fake-time harness for decay/digest/expiry.
- Store-review dry run: MV3 permission audit (`tabs`, `sessions`, `alarms`, `storage`, `notifications` — nothing more; justify each in PRIVACY.md).

## 11. Risks & open questions

- MV3 service-worker eviction bugs are the classic extension trap → event-sourced ledger + alarms design from day 1, chaos tests in CI.
- Users fear auto-closing → default path is confirm-mode until trust is earned (setting suggests silent mode after 2 clean weeks); 30-s undo everywhere; "never delete silently" is a product law.
- Browser API drift / store policy churn → WXT abstraction + minimal permission surface keeps review risk low.
