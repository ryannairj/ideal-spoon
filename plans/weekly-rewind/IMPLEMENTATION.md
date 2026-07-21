# Implementation Spec — Weekly Rewind

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Rewind** (`rewind` binary) · License MIT · Go 1.23 single binary |
| Data dir | `~/.rewind/` — `rewind.db` (SQLite FTS5), `audit/llm-YYYY-MM.jsonl`, `config.toml` |
| Web UI | embedded React at `127.0.0.1:8177` (localhost bind hard-coded; no flag to expose — privacy covenant) |
| Timers | `rewind daemon` for launchd/systemd (generators via `rewind install-service`); collectors hourly, synthesis Friday 16:00 local |
| Privacy modes | `local` (Ollama), `redacted-cloud` (DEFAULT), `full-cloud`; mode per config; `redacted-cloud` sends ONLY: per-project deterministic fact rollups (counts, repo names if allowlisted else `project-N`, commit-subject KEYWORDS (top nouns via local tokenizer), meeting count/hours, domain-cluster topic labels from an allowlist taxonomy) — never raw titles/urls/messages |
| LLM | `packages`-equivalent `internal/llm`: anthropic + openai_compat + ollama |
| Collectors (MVP 6) | git, github, calendar (ICS/CalDAV), browser (chrome/firefox/safari), shell (zsh/bash/fish), note |
| Browser read | copy DB file to temp then SQLite-read (lock avoidance); per-browser path table + version sniffing |
| Projectizer | matchers: repo name; calendar regex; domain groups; sticky corrections in `projects.matchers_json` |
| Citation rule | every narrative bullet carries `[e:id,…]`; validator rejects uncited → 1 regenerate → fallback to deterministic fact list for that section (see §9 for the uncitable-bullet rule) |
| Time estimates | event-density buckets (15-min bins with any activity) + calendar busy blocks; ALWAYS rendered "≈ Nh" |
| Reflection questions | rotating fixed list of 6 in `internal/synth/questions.go` |
| Compile themes | `achievements` (shipped-class only), `learning` (reading+notes), `clients` (per-project rollup) |

## 1. Repository layout

```
rewind/
  cmd/rewind/main.go        # daemon, collect, note, week, compile, audit-llm, doctor, wipe, install-service
  internal/
    collect/{collector.go (iface: Collect(since) ([]Event, error)), git.go, github.go,
             calendar.go, browser/{chrome.go, firefox.go, safari.go, paths.go},
             shell.go, note.go}
    events/{store.go, dedupe.go (hash = sha1(source|kind|title|at))}
    project/{cluster.go, matchers.go}
    facts/{rollup.go}                 # deterministic per-project weekly facts
    synth/{pipeline.go, redact.go (mode boundary), prompts.go, citations.go, questions.go}
    redactrules/{rules.go}            # user regex drop/mask, applied per §0 semantics
    llm/{client.go, anthropic.go, openai.go, ollama.go, audit.go}
    web/{server.go, ui embed}         # timeline, week view, archive, search, settings
    store/{schema.sql}
  fixtures/{week1/ (200-event synthetic dataset), browsers/ (db files per version), llm/}
```

## 2. Dependencies

Go: `modernc.org/sqlite, go-git/go-git (NO — decision: shell out to git CLI for log, simpler+faster), google/go-github/v60, emersion/go-ical, mattn? no; chi, zerolog, BurntSushi/toml, robfig/cron/v3`. UI: react, tanstack-query, tailwind, uplot (time map). Dev: testify, `insta`-style golden via committed files.

## 3. Config (`config.toml`)

```toml
mode = "redacted-cloud"     # local | redacted-cloud | full-cloud
[llm] provider="anthropic" model="claude-sonnet-5" api_key_env="ANTHROPIC_API_KEY"
[llm.local] base_url="http://localhost:11434" model="llama3.3"
[git] roots=["~/code"] identities=["me@x.com","other@y.com"]
[github] token_env="GITHUB_TOKEN" user="me"
[calendar] ics_urls=["https://…"]
[browser] enabled=["chrome","firefox"]  [shell] enabled=true history_files=[]
[redact] rules=[{pattern="(?i)bank", scope="title", action="drop"},
                {pattern="AcmeCorp", scope="all", action="mask"}]
[allowlist] repo_names_to_cloud=true    # false → project-N pseudonyms
[schedule] synth="FRI 16:00" reflect_notify=true
```

## 4. DB schema

```sql
CREATE TABLE events (id INTEGER PK, at INTEGER, source TEXT, kind TEXT, title TEXT,
  detail TEXT, url TEXT, repo TEXT, project_id INTEGER, hash TEXT UNIQUE, raw TEXT);
CREATE VIRTUAL TABLE events_fts USING fts5(title, detail, content=events);
CREATE TABLE projects (id INTEGER PK, name TEXT, matchers_json TEXT, color TEXT, archived INTEGER DEFAULT 0);
CREATE TABLE rewinds (id INTEGER PK, week_start TEXT UNIQUE, doc_md TEXT, doc_json TEXT,
  reflection_md TEXT, generated_at INTEGER, model_mode TEXT);
CREATE TABLE collect_state (source TEXT PK, cursor TEXT, last_ok INTEGER, last_error TEXT);
```

## 5. Synthesis pipeline (`synth/pipeline.go`)

```
friday: facts.rollup(week) per project (pure Go, tested):
  {commitsByRepo, prsMerged[], prsReviewed[], meetings{count, ≈hours, topRecurring[]},
   readingClusters[{topicLabel, topDomains, count}], shellHighlights, switches, deepestFocusBin}
→ redact.ForMode(mode, facts) → payload
→ narrative prompt (facts + event-id map) → sections JSON {narrative[], shipped[], inflight[],
   collab[], reading[], context{switches, focus}} each bullet {text, cites:[eventIds]}
→ citations.Validate (ids exist ∧ belong to week) → regenerate once → degrade section to facts
→ render md + json → store → notify (desktop notification via beeep) → web /week/<date>
Sections "Shipped" come from deterministic facts only (merged PRs, tags, deploy-pattern commits)
— LLM orders/phrases but cannot add items (validator: shipped bullets map 1:1 to fact entries).
```

## 6. CLI/API surfaces

`rewind note "text"` · `rewind week [--date]` (render to stdout) · `rewind compile --since 2026-01-01 --theme achievements [--out brag.md]` · `rewind audit-llm [--last]` (prints exact outbound payloads from audit JSONL) · `rewind doctor` (permissions: browser file access/TCC, ics reachability, git roots) · `rewind wipe --source browser|--all`. Web (localhost): `/` timeline (filter source/project), `/week/<date>` rewind + reflection box, `/archive` calendar, `/search`, `/settings` (projects matcher editor, redact rules tester: paste string → shows drop/mask result).

## 7. Milestone task lists

**M0** — T1 scaffold + schema + config + doctor; T2 git collector (roots walk, multi-identity log parse) + note + dedupe; T3 raw week view (terminal `rewind week --raw` + web timeline); T4 CI + release builds. *Useful log ships.*
**M1** — T1 github + calendar collectors (cursor state, failure → warning event); T2 browser readers ×3 (fixtures per schema version; copy-then-read; locked/missing graceful); T3 shell (dedupe + noise filter list); T4 projectizer + web matcher editor + stickiness tests; T5 FTS search.
**M2** — T1 facts.rollup + golden tests on `fixtures/week1`; T2 redact.ForMode + **boundary test: in redacted-cloud, outbound payload fixture contains zero raw titles/urls (string-set intersection empty)** — release gate; T3 narrative synthesis + citations validator (+unit suite) + shipped-1:1 rule; T4 local-mode (Ollama) full path test; T5 week page render + notification.
**M3** — T1 reflection capture (rotating question, skip-safe) + archive; T2 redact rules engine (drop pre-storage, mask pre-LLM — semantics tests) + settings tester UI; T3 audit-llm (byte-for-byte vs request log test); T4 wipe per source.
**M4** — T1 compile themes (citation reuse, 12-week < 3 min bench) + brag-doc output; T2 encrypted export/import (`age` passphrase); T3 packaging (brew formula, deb, install-service) + LOAM-style tone pass on prompts (observational, kind — checklist doc); T4 README covenant section ("never a team surveillance tool" + localhost-only statement). Tag v1.0.

## 8. Test mapping

Deterministic layers (collectors→facts) fully golden-tested on synthetic week. Privacy boundary + audit fidelity = release gates. Recorded-LLM narrative tests; nightly live smoke in all 3 modes (local via CI Ollama container).

## 9. Edge-case semantics

**Citation validator — uncitable bullets (exact rule).** `citations.Validate` runs per section:

```
for each section:
  bad = bullets where cites is empty OR any id ∉ thisWeek.eventIds
  if bad is empty: keep section as-is
  else:
    regenerate section once (prompt includes the offending bullets + "every bullet MUST cite")
    re-validate:
      if now clean: keep
      else: DROP the still-uncited bullets; if that empties the section,
            replace the whole section with the deterministic fact list for that section
```

So a single uncitable bullet is dropped (not the whole section); only a fully-uncitable section degrades to facts. AC: a narrative with one hallucinated bullet renders the rest + drops that bullet; a wholly-uncited section renders the fact list.

**Mask determinism for FTS.** Masking is applied deterministically **before storage** for `drop` rules and **before LLM** for `mask` rules — but the FTS index is built from stored events, which retain original text (local-only DB, privacy covenant permits this since it never leaves the machine). Masking is therefore never applied to the FTS content, so search stays consistent and complete. The redact boundary only governs what crosses the LLM/cloud boundary (tested by the §7 M2 boundary test), not what the local index contains. AC: searching a masked term still finds the local event; the cloud payload fixture contains zero occurrences of it.

**Project matcher stickiness scope.** Rewind is a single-user local tool, so "user scope" = the whole install. A sticky correction (user manually reassigns an event's project) is stored in `projects.matchers_json` as an explicit event-hash pin and **always wins** over regex/domain matchers, globally and for future weeks, until removed. AC: reassigning one event keeps every matching future event on that project.

**Time-estimate bin boundaries (non-overlapping).** 15-minute bins are half-open `[binStart, binStart+15m)` aligned to the local-time hour; an event at exactly a boundary belongs to the **later** bin. A bin counts as active if ≥1 event falls in it; `≈ hours = activeBins × 15m` rounded to the nearest 0.5 h. Calendar busy blocks union with activity bins (no double-count of an overlapping bin). AC: an event at `10:15:00` lands in the `10:15` bin, not `10:00`.

**Browser schema-version coverage (enumerated).** `browser/paths.go` carries an explicit table; unknown/newer versions fall back to the latest known schema for that browser and emit a warning event rather than failing:

| Browser | DB file | Known schema range (MVP) | Unknown-version behavior |
|---|---|---|---|
| Chrome/Chromium | `History` (SQLite) | Chrome 90–current `urls`/`visits` layout | try latest-known query; warn event on error |
| Firefox | `places.sqlite` | `moz_places`/`moz_historyvisits` (FF 80+) | same fallback + warn |
| Safari | `History.db` | `history_items`/`history_visits` (macOS 11+) | same fallback + warn; TCC-denied → doctor flag |

Fixtures under `fixtures/browsers/` include one DB file per listed schema; the collector is tested against each and against a synthetic "unknown version" file to prove graceful degradation.
