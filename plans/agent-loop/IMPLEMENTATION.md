# Implementation Spec — Loam (agent loop)

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Loam** (`loam` CLI/daemon) · License MIT |
| Language | Python 3.12, `uv`, `ruff`, `pytest`; package `loam/` |
| LOC budget | core (`loam/core/**`) ≤ 3,000 lines excl. tests/tools — CI check via `scripts/loc_budget.py` |
| LLM | own thin client (httpx): `anthropic`, `openai_compat` (covers Ollama); NO framework deps (no langchain etc.) |
| DB | SQLite `~/.loam/loam.db` (stdlib `sqlite3`); audit JSONL `~/.loam/audit/YYYY-MM-DD.jsonl` |
| Config | `~/.loam/config.toml` (provider, persona, surfaces) + `~/.loam/grants.toml` (capabilities), both hot-reloaded (mtime check per turn) |
| Loop limits | max 8 tool iterations/turn; per-tool timeout 30 s; context: persona + grants summary + top-5 memories + last 20 turns |
| Embeddings | `fastembed` (local, default) or provider; memory retrieval hybrid: cosine + SQLite FTS5, RRF |
| Scheduler | APScheduler 4, SQLite jobstore; missed-fire grace 12 h |
| Surfaces | CLI (typer + rich), Telegram (`python-telegram-bot` 21), web (FastAPI + Jinja, read-mostly, `127.0.0.1:8990`) |
| Tools (MVP 6) | `memory`, `schedule`, `web_fetch`, `rss`, `weather` (open-meteo, no key), `shell` |
| Scopes | `memory:read|write`, `schedule:manage`, `web:fetch[:domain]`, `rss:read`, `weather:read`, `shell:<binary>` |
| Risky-scope confirm | scopes listed in `grants.toml [confirm]` require per-invocation yes via active surface (60 s timeout → deny) |
| Time parsing | `dateparser`, always confirm-echo parsed result |

## 1. Repository layout

```
loam/
  pyproject.toml
  loam/
    core/                      # the ≤3k-line readable core
      loop.py                  # ~200 lines: assemble → model → tools → reply
      context.py  bus.py       # message bus between surfaces & loop
      llm.py                   # both providers + streaming
      tools/base.py            # Tool protocol: name, scopes, schema, run()
      grants.py                # parse grants.toml, check(scope), hot-reload
      audit.py                 # append JSONL + sqlite index; every model/tool call
      memory.py  scheduler.py  store.py (sqlite schema+dao)
    tools/{memory_tool.py, schedule_tool.py, web_fetch.py, rss.py, weather.py, shell.py}
    surfaces/{cli.py, telegram.py, web/{app.py, templates/}}
    main.py                    # loam chat|ask|daemon|audit|verify|wipe
  tests/{unit/…, injection/shell_injection_test.py, soak/}
  docs/LOAM_TOUR.md
```

## 2. Dependencies (pyproject)

`httpx, typer, rich, python-telegram-bot, fastapi, uvicorn, jinja2, apscheduler>=4, dateparser, fastembed, feedparser, tomlkit, pydantic` (tool schemas). Dev: `pytest, pytest-asyncio, ruff, freezegun`.

## 3. Config formats

```toml
# config.toml
[llm] provider="anthropic" model="claude-sonnet-5" api_key_env="ANTHROPIC_API_KEY"
[persona] name="Loam" style="concise, warm"  [web] enabled=true port=8990
[telegram] enabled=true token_env="TELEGRAM_TOKEN" allowed_chat_ids=[123]
# grants.toml
[grants] "memory:read"=true "memory:write"=true "schedule:manage"=true
"web:fetch"= ["*.wikipedia.org", "news.ycombinator.com"]   # list = domain allowlist; true = any
"rss:read"=true "weather:read"=true
"shell:ls"=true "shell:df"=true
[confirm] scopes=["shell:*"]        # per-invocation confirmation
[trusted] scopes=[]                 # confirm-exempt overrides
```

## 4. DB schema

```sql
CREATE TABLE turns (id INTEGER PK, surface TEXT, direction TEXT CHECK(direction IN ('in','out')),
  content TEXT, at INTEGER);
CREATE TABLE memories (id INTEGER PK, text TEXT NOT NULL, source_turn_id INTEGER,
  tags TEXT DEFAULT '[]', embedding BLOB, superseded_by INTEGER, created_at INTEGER);
CREATE VIRTUAL TABLE memories_fts USING fts5(text, content=memories);
CREATE TABLE schedules (id INTEGER PK, kind TEXT CHECK(kind IN ('reminder','routine')),
  spec TEXT, payload TEXT, enabled INTEGER DEFAULT 1, last_run_at INTEGER);
CREATE TABLE audit_idx (id INTEGER PK, at INTEGER, kind TEXT, session TEXT,
  tokens_in INTEGER, tokens_out INTEGER, cost REAL, jsonl_file TEXT, jsonl_offset INTEGER);
```

## 5. Core loop contract (`loop.py`)

```
handle(msg: InboundMsg) -> reply:
  reload configs if mtime changed
  tools = [t for t in registry if grants.allows_any(t.scopes)]
  ctx = persona + tool list + retrieve_memories(msg.text, k=5) + recent turns
  for i in range(8):
    resp = llm.chat(ctx, tools)
    if resp.tool_calls:
      for call in resp.tool_calls:
        if not grants.check(call.scope_instance): audit('tool_denied'); feed denial to model; continue
        if needs_confirm(call): ok = surface.confirm(call.summary, 60s); if not ok: audit+feed denial
        result = run_with_timeout(call, 30s)   # exceptions → structured error fed back ONCE
      continue
    return resp.text
  return honest_failure()
Every llm.chat and tool run emits audit rows (invariant enforced by wrapper — there is no unaudited path).
```
Shell tool: argv-only execution (`subprocess.run(list, shell=False)`), binary must match a `shell:<binary>` grant, args validated against metacharacter denylist even though shell=False (defense in depth), cwd `~`, output cap 8 KiB.

## 6. Surfaces & commands

CLI: `loam chat` (REPL), `loam ask "…"`, `loam daemon` (telegram+web+scheduler), `loam audit [--since] [--kind]`, `loam verify` (LOC budget, grants sanity, db integrity), `loam wipe --memories|--turns`. Telegram: allowlist check first (silent drop + audit for strangers); confirm prompts as inline keyboard yes/no. Web pages: `/audit` (filterable table), `/memory` (list, forget buttons), `/grants` (read-only render of toml), `/costs` (monthly rollup).

## 7. Milestone task lists

**M0** — T1 package scaffold + store + audit (JSONL+idx, replay-rebuild test); T2 llm.py both providers + streaming CLI chat; T3 loop.py + tools/base + registry; T4 CI (ruff, pytest, LOC budget script). *Readable core ships.*
**M1** — T1 grants.py + hot-reload + denial paths; T2 memory tool + retrieval (hybrid RRF) + forget/supersede + eval fixture (30 memories/15 queries ≥85% top-3); T3 web_fetch (GET only, 1 MB cap, domain allowlist, readability extract via `trafilatura`) + rss + weather; T4 injection suite: shell.py refuses `;`, `&&`, backticks, `$()`, pipes, glob-expansion tricks, PATH tricks (release-blocking).
**M2** — T1 scheduler (reminders: dateparser + confirm-echo; routines: cron + prompt template through same loop with surface=scheduler); T2 missed-fire grace + DST tests (freezegun, tz from config); T3 telegram surface + allowlist + confirm keyboards; T4 long-output truncation + web-UI link.
**M3** — T1 FastAPI web (4 pages) + cost rollups; T2 shell tool behind [confirm] default; T3 LOAM_TOUR.md (file-by-file walkthrough, kept ≤ core reading time 1 h); T4 48 h soak script (`tests/soak/`, memory-leak + schedule-drift asserts) run in nightly CI; T5 packaging (uv tool install, systemd unit example). Tag v1.0.

## 8. Test mapping

Release-blocking: injection suite, audit-invariant test (grep loop for uninstrumented paths + runtime assertion), allowlist-stranger test (zero-information reply). Recorded-LLM fixtures for loop unit tests; nightly live smoke. LOC budget CI gate.
