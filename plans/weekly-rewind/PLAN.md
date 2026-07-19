# Weekly Rewind — the journal that writes itself from your week's exhaust

**Category:** Life improvement / dev tool crossover · **Difficulty:** Medium · **Platform:** Local-first CLI/daemon + web view

## 1. Vision

You did things this week — shipped code, sat in meetings, read stuff, went places — but by Friday it's a blur, and by review season it's gone. Weekly Rewind quietly collects your activity exhaust (git commits, calendar events, browser history, shell history, optionally more), and every Friday writes an honest, well-organized rewind: what you worked on, what actually shipped, where the time went, what you kept context-switching between — plus a 60-second prompt for the one thing machines can't capture: how it felt. Brag-doc, standup fuel, and memory prosthetic in one.

## 2. Why it can win

- Journaling fails because it demands input at the worst time; automatic tools (RescueTime) give charts, not *narratives*. The wedge: **local-first collection + LLM narrative synthesis with citations** — every claim in the rewind links to the commits/events behind it.
- Privacy-first architecture (everything stays on your machine; LLM calls default to redacted summaries or a local model) makes a category that's creepy-as-SaaS acceptable as a tool.

## 3. Users & use cases

Developers (perf reviews, standups), consultants (client time recall), anyone doing weekly reviews.

1. Friday 16:00 notification → open rewind: narrative + "shipped" list + time map + context-switch score → answer one reflective question → archived.
2. Perf review: `rewind compile --since january --theme achievements` → brag-doc draft with links to every PR.
3. Monday standup: last week's "shipped + in-flight" section, copy-paste ready.
4. "What was that library I was reading about two weeks ago?" → search across collected traces.

## 4. MVP scope & non-goals

**MVP:** collectors — git (all local repos, multi-remote identity mapping), GitHub API (PRs/issues/reviews), calendar (ICS URL / CalDAV), browser history (Chrome/Firefox/Safari local DB read), shell history (zsh/bash/fish, opt-in), manual notes (`rewind note "…"`); weekly synthesis with citations; reflection prompt; web view (localhost) with timeline + search; compile mode (multi-week rollups); redaction rules; export markdown.
**Non-goals:** screenshots/screen recording (too creepy, wrong ROI), keystroke/app trackers, team/manager dashboards (explicitly never — this is a personal tool, stated in the README as a covenant), cloud sync in MVP (encrypted export/import instead), mobile.

## 5. Tech stack

- **Core:** Go 1.23 single binary (`rewind`): daemon mode (launchd/systemd timers) + CLI; SQLite (FTS5) at `~/.rewind/`; embedded web UI (localhost only).
- **LLM:** provider-agnostic; three privacy modes — `local` (Ollama), `redacted-cloud` (default: only derived summaries/counts leave, raw titles/urls/messages never sent unless allowlisted), `full-cloud` (opt-in).
- **Collectors:** each an isolated module implementing `Collect(since) []Event`; failures degrade to "source unavailable this week" — never block the rewind.

## 6. Architecture

```
timers → collectors → events(normalized: at, source, kind, title, detail, refs, project_hint)
             → projectizer: cluster events → projects (repo names, calendar patterns, domain groups; user-correctable, sticky)
Friday → synthesizer: per-project facts (deterministic rollups: commits, PRs merged, meetings, reading clusters, hours-ish)
       → LLM narrative over facts (citations = event ids; validator rejects uncited claims)
       → rewind doc (md + json) → notification → reflection capture → archive
```

Time accounting is honest-by-design: derived from event density/calendar blocks, always labeled "estimated"; no fake minute-precision.

## 7. Data model (SQLite)

- `events(id, at, source{git,github,cal,browser,shell,note}, kind, title, detail, url, repo, project_id NULLABLE, hash UNIQUE /*dedupe*/, raw_json)`
- `projects(id, name, matchers_json /*repos, cal regex, domains*/, color, archived)`
- `rewinds(id, week_start, doc_md, doc_json /*sections with citation ids*/, reflection_md, generated_at, model_mode)`
- `redactions(id, pattern, scope{title,url,detail}, action{drop,mask}) ` · `settings(key, value)`

## 8. Feature specs

**F1 — Collectors.** Git: walk configured roots for repos, `git log --author=<identities>` since last run; GitHub: authored/reviewed/merged via token; Calendar: ICS/CalDAV pull, busy-block extraction; Browser: read history DBs (documented per-browser paths, file-copy-then-read to avoid locks), cluster by domain+time into "reading sessions"; Shell: dedup + noise-filter (cd/ls stripped). *AC:* each collector has fixture-based tests; a locked/missing source yields a visible warning event, not a crash; dedupe = re-running collectors never duplicates (hash test).

**F2 — Projectizer.** Auto-clusters with confidence; UI/CLI reassign ("this domain → Project X") persists as matcher. *AC:* corrections are sticky across weeks; unclustered events land in "Misc" (never dropped).

**F3 — Weekly synthesis.** Sections: Narrative (5–10 sentences), Shipped (merged PRs/tagged releases/deploy-ish commits), In flight, Meetings & collaboration (hours-est, top recurring), Reading & research (clustered topics w/ top links), Context report (project-switch count, deepest focus block), all claims cited (hover/click → source events). *AC:* citation validator: 100% of factual bullets carry ≥ 1 event id (uncited → regenerate or degrade to raw fact list); generation < 2 min; works fully in `local` mode with an Ollama model.

**F4 — Reflection & archive.** One rotating question (what energized you / drained you / one thing to change); 60-second capture (text or skip); archive view = calendar of weeks; streaks shown gently (no guilt cop). *AC:* skipping reflection never blocks archiving.

**F5 — Search & compile.** FTS across events + rewinds; `rewind compile --since <date> [--theme achievements|learning|clients]` → rollup doc reusing weekly citations. *AC:* compile of 12 weeks < 3 min; brag-doc theme lists only `shipped`-class items with links.

**F6 — Privacy controls.** Redaction rules (regex on titles/urls, e.g. drop `*bank*`, mask client names); per-source pause; `rewind audit-llm` prints exactly what was sent to any cloud model last run; panic command `rewind wipe --source browser`. *AC:* redaction applied before storage for `drop`, before LLM for `mask` (tests); audit-llm output matches actual request logs byte-for-byte.

## 9. Milestones

- **M0 (1):** binary + SQLite schema + git & note collectors + raw week view (no LLM). *Ships: useful log immediately.*
- **M1 (2):** GitHub + calendar + browser + shell collectors; projectizer; timeline UI + FTS.
- **M2 (3):** synthesis pipeline + citations + validator + local-model mode. *Ships: the Friday magic.*
- **M3 (4):** reflection loop, notifications, redaction + audit-llm.
- **M4 (5):** compile/brag-doc mode, export/encrypted backup, packaging (brew/deb) + docs. *Ships: v1.0.*

## 10. Testing

- Fixture "synthetic week" dataset (200 events across sources) → deterministic rollup snapshot tests + recorded-LLM narrative tests; citation validator unit suite.
- Collector fixtures per browser/shell format version; corrupted-DB and permission-denied paths.
- Privacy tests are release-blocking: redaction, mode boundaries (in `redacted-cloud`, assert no raw title strings appear in outbound payload fixtures).

## 11. Risks & open questions

- Browser history DB formats shift → version-sniffing readers + fixture per version; failure = skip source gracefully.
- The narrative could feel like surveillance-of-self → tone guidelines (observational, kind, zero productivity-shaming) baked into the prompt pack and reviewed; time numbers always "≈".
- macOS TCC permissions for reading browser files → guided one-time setup with checks (`rewind doctor`).
