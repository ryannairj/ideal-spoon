# Implementation Spec — TabSweep

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **TabSweep** · License MIT |
| Toolkit | WXT + TS 5 + React 18; targets: `chrome-mv3` (M0–M3), `firefox-mv3` (M4); Manifest permissions EXACTLY: `tabs, storage, alarms, notifications, sessions, unlimitedStorage` (each justified in PRIVACY.md; NO host permissions — no content scripts needed) |
| Storage | Dexie 4 (IndexedDB) in extension context; `chrome.storage.local` only for tiny settings mirror the service worker needs synchronously |
| Alarms | `decay-eval` hourly; `digest` weekly (user day/hour, default Fri 16:00); `expiry-sweep` daily |
| Tab identity | `windowId:tabId` while live; on restart reconcile by (url, windowIndex, tabIndex) heuristic; unresolved → treat as new (age preserved via url-hash match if unique) |
| Defaults | TTL 14 d; sweep style: confirm-batch daily until user enables silent (suggested after 14 d clean); shelf expiry 60 d + 7 d grace; aging cue OFF |
| Protected (never auto-swept) | pinned, audible, active-in-any-window, whitelisted domain/window, form-activity flag |
| Form activity heuristic | tab has `status=complete` and title changed after user input? Not detectable without content scripts — decision: use `chrome.tabs.Tab.autoDiscardable === false` OR url matches `/checkout|compose|edit|draft|form/` pattern list; imperfect, biased to protect |
| Intent categories | `read, buy, reference, reply, watch, unknown`; heuristics: domain packs `content/domains.json` + url/title regex pack `content/patterns.json` (versioned, in-repo) |
| Topic clusters | local: normalized title token TF-IDF + greedy centroid clustering (no embeddings dependency); cluster label = top 2 tokens |
| LLM (optional) | BYO key (openai_compat incl. Ollama URL); batch: `[{url, title}] → [{intent, topic}]` 50/req; results cached by url-hash; NEVER page content |
| Digest highlights | form-activity tabs, work-domain (top-5 frequent domains) docs, re-opened-after-sweep urls |
| Import/export | OneTab format parser; export: JSON (full) + Netscape bookmarks HTML |

## 1. Repository layout

```
tabsweep/
  entrypoints/
    background.ts               # event wiring only; logic in modules
    popup/{main.tsx, Popup.tsx}          # quick stats + sweep-now + open shelf
    shelf/{main.tsx, Shelf.tsx}          # full-page app (chrome-extension://…/shelf.html)
    digest/{main.tsx, Digest.tsx}
    options/{main.tsx, Options.tsx}
    onboarding/{main.tsx, Amnesty.tsx}
  src/
    ledger/{ledger.ts (event-sourced ops), reconcile.ts, ageing.ts}
    sweep/{rules.ts (protection table), executor.ts, undo.ts (30 s toast state)}
    shelf/{store.ts, lifecycle.ts (state machine), expiry.ts, search.ts (minisearch)}
    categorize/{heuristics.ts, cluster.ts, llm.ts, corrections.ts}
    digest/{build.ts, highlights.ts}
    io/{onetab.ts, exportJson.ts, exportHtml.ts}
    db.ts (dexie schema)  settings.ts  messages.ts (typed runtime messages)
  content/{domains.json, patterns.json}
  fixtures/{tabs-labeled.json (500), onetab-export.txt, ledger-scenarios/}
  e2e/ (playwright + chromium w/ extension)
  PRIVACY.md
```

## 2. Dependencies

`wxt, react, dexie, minisearch, zod, tailwindcss`. Dev: vitest (+`@webext-core/fake-browser` for unit tests of background logic), playwright. No runtime network deps besides optional LLM fetch.

## 3. Dexie schema

```ts
tab_ledger: 'tabKey, urlHash, lastActive'   // {tabKey, url, urlHash, title, favicon, firstSeen,
                                            //  lastActive, pinned, audible, windowLabel, protectedReason?}
shelf: 'id, state, sweptAt, intent, topic'  // {id, url, urlHash, title, favicon, sweptAt, source,
                                            //  intent, topic, note?, state, expireAt, lastTouched}
rules: '++id, kind'                          // whitelist_domain | whitelist_window | ttl_override | never_expire
digests: 'week'                              // {week, stats, highlights, reviewedAt}
llm_cache: 'urlHash'                         // {urlHash, intent, topic, model, at}
corrections: 'domain'                        // {domain, intent}  — user overrides, rank 1
```

## 4. Background logic contracts

**Ledger:** listeners (`onActivated, onUpdated, onRemoved, onCreated, windows.onFocusChanged`) append ops to an in-memory buffer flushed to Dexie every 30 s AND on `runtime.onSuspend`; hourly alarm re-walks `chrome.tabs.query({})` to self-heal drift (authoritative reconcile). MV3-kill resilience: all state derivable from Dexie + live query; chaos test terminates worker mid-cycle.
**Decay eval (hourly):** stale = `now - lastActive > TTL` ∧ not protected (rules.ts table — every clause unit-tested); confirm-mode → add to pending batch (badge count, popup lists it); silent-mode → sweep immediately with 30 s undo toast (notification with "Undo" button).
**Sweep executor:** for batch: write shelf rows FIRST (source, categorize sync-heuristics inline), then `chrome.tabs.remove` — crash between = duplicate shelf row max, never lost tab (ordering test); restore = `chrome.tabs.create` (+ `sessions.restore` where available for scroll).
**Categorize:** heuristics at sweep time; llm.ts batch pass on alarm if key configured (uncategorized-only); corrections override both.

## 5. UI contracts

**Amnesty (first run):** scan stats spectacle → age slider (preview count updates live) → whitelist quick-picks (top-10 domains by tab count, work-domain suggestions) → single confirm → sweep → celebration + "nothing deleted, it's all on the shelf" reassurance. < 90 s scripted E2E.
**Shelf:** virtualized list (react-window-less: manual windowing, keep deps minimal); group tabs [Intent | Topic | Week | Domain]; fuzzy search (minisearch, <50 ms @2k test); row: favicon, title, age, intent chip (click-to-correct), actions restore/keep/note/let-go; bulk select; state machine `shelved → kept|restored|let_go → expired` (lifecycle.ts table-tested); grace notice screen lists exactly the expiring set; `never_expire` rule wins (test).
**Digest:** stats header (swept count, browser "weight lost"), highlights sections, let-go pile preview with one-click triage rows, skip-forever setting.
**Popup:** counts (open tabs, pending sweep, shelf), Sweep-now, open shelf/digest.

## 6. Milestone task lists

**M0** — T1 WXT scaffold + permissions manifest + PRIVACY.md; T2 ledger + reconcile + fake-browser unit tests + restart persistence (e2e relaunch test); T3 manual sweep (popup select-all-stale → shelf) + basic shelf list + restore.
**M1** — T1 rules.ts protection table + decay alarm + confirm-batch flow + badge; T2 undo toast + notification button; T3 amnesty flow + E2E; T4 sweep-ordering crash test + 200-tab sweep < 5 s bench (batched `tabs.remove`).
**M2** — T1 shelf UX complete (groups, search, bulk, lifecycle, expiry + grace + never_expire); T2 heuristics.ts + content packs + labeled-fixture metric CI (≥75%); T3 cluster.ts (TF-IDF greedy) + topic groups; T4 corrections (sticky, ranked-first test).
**M3** — T1 digest build + highlights (incl. re-opened-after-sweep tracking via urlHash match on onCreated) + weekly alarm + badge nudge; T2 llm.ts optional pass (+Ollama URL support, urls+titles-only assertion test, cache) → ≥90% on fixtures with LLM; T3 onetab import (1k fixture, import-day dating note) + JSON/HTML export + import-back lossless test.
**M4** — T1 firefox build (WXT target; API diffs behind `browser.` polyfill audit) + parity e2e subset; T2 store listing assets + screenshots script + privacy manifest ("no data leaves your machine unless you add an LLM key — and then only titles/urls"); T3 MV3 permission audit test (manifest declares exactly the §0 set); T4 alarm/fake-time suite polish. Tag v1.0 + store submissions.

## 7. Test mapping

Playwright-with-extension: ledger restart, sweep/undo/restore, amnesty, MV3 worker-kill chaos, digest triage. Vitest: rules table, lifecycle machine, categorize metrics vs `tabs-labeled.json`, io round-trips. "Never delete silently" invariant: grep-level test that no `tabs.remove` call site lacks a preceding shelf write or explicit user action context.
