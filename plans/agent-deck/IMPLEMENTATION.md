# Implementation Spec — Agent Deck

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Names | daemon `deckd`, CLI `deck`, module `github.com/OWNER/agentdeck` |
| License | MIT · Language Go 1.23 (daemon+CLI), React 18+TS PWA (UI) |
| PTY | `github.com/creack/pty`; ring buffer 50k lines (byte-capped 8 MiB) per session |
| DB | SQLite `~/.local/share/agentdeck/deck.db` (`modernc.org/sqlite`) |
| Transport | WebSocket `/ws` (nhooyr `coder/websocket`) multiplexing all sessions; REST for CRUD |
| Listen | default `127.0.0.1:7433`; `--listen` requires `--token-file` or `DECK_TOKEN` + TLS or `--insecure` |
| Session lifetime | sessions die with daemon (MVP decision per PLAN §10); on daemon start, orphan rows marked `exited(unknown)` |
| Adapter packs | JSON files in `packs/` embedded + `~/.config/agentdeck/packs/` override; schema versioned `packVersion: 1` |
| State machine | `starting → working ⇄ waiting_input | waiting_approval → done | exited(code)`; transitions only via detector hits or process events |
| Push | Web Push VAPID (`SherClockHolmes/webpush-go`); keys generated first run |
| Transcript storage | zstd-compressed 64 KiB segments |
| xterm | `@xterm/xterm@5` + fit/webgl/search addons |

## 1. Repository layout

```
agentdeck/
  cmd/deckd/main.go               # serve; flags: --listen, --insecure, --token-file, --satellite (v1)
  cmd/deck/main.go                # run, list, kill, attach-url, token
  internal/
    daemon/{server.go, rest.go, ws.go, auth.go, push.go}
    session/{manager.go, session.go, ptyio.go, ring.go, transcript.go}
    state/{engine.go, detectors.go, approvals.go}   # pack interpreter
    packs/loader.go  packs/data/{claude-code.json, codex.json, opencode.json, fallback.json}
    store/{db.go, migrations/*.sql}
    diff/gitdiff.go                # working-tree diff vs session-start ref
    notify/webpush.go
  ui/                              # Vite React PWA → embedded
    src/
      pages/{Board.tsx, Session.tsx, Settings.tsx}
      components/{SessionCard.tsx, Terminal.tsx, ApprovalSheet.tsx, MobileKeysBar.tsx,
                  DiffView.tsx, StateBadge.tsx, PushSetup.tsx}
      lib/{ws.ts, api.ts, types.ts}
      sw.ts                        # service worker: push handling, PWA
  fixtures/transcripts/{claude-code-v2.x/*.cast, codex/*.cast, opencode/*.cast}
  cmd/fakeagent/main.go            # replays .cast fixtures interactively (test agent)
  .github/workflows/ci.yml
```

## 2. Dependencies

Go: `creack/pty`, `coder/websocket`, `modernc.org/sqlite`, `klauspost/compress/zstd`, `SherClockHolmes/webpush-go`, `alexedwards/argon2id`, `go-chi/chi/v5`, `spf13/cobra`, `rs/zerolog`. UI: `react`, `@xterm/xterm`+addons, `zustand`, `react-router-dom`, `tailwindcss`, `diff2html` (DiffView), `vite-plugin-pwa`. Dev: `vitest`, `playwright`.

## 3. Configuration

`~/.config/agentdeck/config.toml`: `listen`, `token_file`, `notify {quiet_hours = "22:00-07:00"}`, `packs_dir`. `deck` finds daemon via `DECK_ADDR` or default. `deck run` auto-starts daemon if absent (unix socket handshake first, then spawn).

## 4. Database schema

```sql
CREATE TABLE sessions (id TEXT PRIMARY KEY, host_id TEXT NOT NULL DEFAULT 'local',
  cwd TEXT NOT NULL, repo_root TEXT, git_branch TEXT, start_ref TEXT,
  cmd TEXT NOT NULL, adapter TEXT NOT NULL, state TEXT NOT NULL,
  title TEXT, last_line TEXT, cost_line TEXT,
  started_at INTEGER NOT NULL, ended_at INTEGER, exit_code INTEGER, pinned INTEGER DEFAULT 0);
CREATE TABLE events (id INTEGER PRIMARY KEY AUTOINCREMENT, session_id TEXT NOT NULL,
  at INTEGER NOT NULL, kind TEXT NOT NULL CHECK(kind IN
    ('state_change','approval_request','approval_answer','notice','user_input','write_audit')),
  payload TEXT NOT NULL);
CREATE INDEX events_session ON events(session_id, id);
CREATE TABLE transcripts (session_id TEXT NOT NULL, seq INTEGER NOT NULL,
  bytes BLOB NOT NULL, PRIMARY KEY(session_id, seq));
CREATE TABLE push_subs (id TEXT PRIMARY KEY, endpoint TEXT UNIQUE, keys_json TEXT,
  filters_json TEXT DEFAULT '{}');
CREATE TABLE auth_tokens (id TEXT PRIMARY KEY, name TEXT, token_hash TEXT, created_at INTEGER);
CREATE TABLE hosts (id TEXT PRIMARY KEY, name TEXT, token_hash TEXT, last_seen INTEGER); -- v1
```

## 5. Contracts

**REST (`/api`, bearer token when non-local):** `GET /sessions` (cards: id, adapter, state, durations, repo/branch, last_line, needs_attention sort key) · `GET /sessions/{id}` · `DELETE /sessions/{id}` (SIGTERM→KILL after 5 s) · `GET /sessions/{id}/diff` → `{files:[{path, status, patch}]}` · `GET /sessions/{id}/transcript?from_seq=` · `POST /push/subscribe` · `GET /vapid.pub` · `GET /healthz`.

**WS protocol (JSON frames):**
```
client→server: {t:'sub', sessionId} | {t:'unsub'} | {t:'stdin', sessionId, dataB64}
              | {t:'resize', sessionId, cols, rows} | {t:'approve', sessionId, requestId, optionKey}
server→client: {t:'out', sessionId, dataB64} | {t:'state', sessionId, state, since, meta}
              | {t:'approval', sessionId, requestId, prompt, options:[{key,label,keystrokes}], confidence}
              | {t:'sessions_delta', card} | {t:'gone', sessionId}
```
`approve` maps optionKey → keystrokes from the approval event; server re-validates the approval is still pending (else error frame) — never blind-sends keys.

**`deck run`:** `deck run [--title t] -- <cmd…>` → registers session (adapter = pack match on argv[0], else `fallback`), fork PTY, mirror stdio (transparent passthrough), stream to daemon over local WS. Overhead requirement: writes are unbuffered passthrough; measure in bench test.

## 6. Adapter pack schema (`packVersion: 1`)

```jsonc
{
  "packVersion": 1, "name": "claude-code", "matchArgv0": ["claude"],
  "detectors": [
    {"on":"output","regex":"(?m)^.*Esc to interrupt.*$","set":"working"},
    {"on":"output","regex":"Do you want to .*\\?","set":"waiting_approval","capture":"approval"},
    {"on":"silence","afterMs":15000,"ifStateIn":["working"],"set":"waiting_input"},
    {"on":"output","regex":"\\$ $","set":"waiting_input"}
  ],
  "approvalParser": {
    "promptRegex": "(?s)(Do you want to [^\\n]+)\\n(?<opts>(\\s+\\d+\\..*\\n)+)",
    "optionRegex": "\\s+(?<key>\\d+)\\.\\s+(?<label>.+)",
    "answerKeystrokes": "{key}\\r",
    "minConfidence": 0.9
  },
  "titleRegex": "…", "costLineRegex": "…tokens.*$"
}
```
Engine rules: detectors evaluated on ANSI-stripped output (keep raw for terminal); approval confidence = parser matched prompt AND ≥ 2 options AND options contiguous — else emit `approval_request` with `confidence:0` (UI shows raw terminal, buttons disabled).

**Pack versioning strategy.** `packVersion` is a single integer (not SemVer) — it names the *pack schema contract*, not the pack's content. Rules:

- The loader supports exactly the current `packVersion` (MVP: `1`). A pack whose `packVersion` is unknown/higher → skipped with a logged warning (`deck packs lint` flags it); the built-in `fallback.json` still covers that agent.
- Editing an existing pack's regexes/labels is **not** a version bump — pack content is expected to drift as agents change their output (that's why fixtures, not code, gate packs; see §9). Content changes ship freely; only a breaking *schema* change (new required field, changed field meaning) increments `packVersion`, and the loader would then carry a small migration for older embedded packs.
- User override packs in `~/.config/agentdeck/packs/` win over embedded packs of the same `name`; a `packVersion` mismatch on an override is a hard error surfaced in Settings (so a stale hand-written pack fails loudly rather than silently misparsing).

## 7. UI screens

`Board` — responsive card grid, needs-attention first (waiting_approval > waiting_input > working > done/exited), pull-to-refresh, state badges with live durations. `Session` — tabs: Terminal (xterm + MobileKeysBar: Esc/Tab/Ctrl/arrows/Enter), Approval sheet (bottom sheet when pending: prompt text + option buttons + "show terminal" fallback), Diff, Info (events timeline). `Settings` — push setup, quiet hours, tokens, packs list.

## 8. Milestone task lists

**M0** — T1 daemon skeleton + store + REST list; T2 `deck run` PTY wrap + passthrough + registration; T3 WS out-streaming + single-session Terminal page; T4 fakeagent binary replaying .cast fixtures; T5 CI + passthrough bench (< 5 ms added latency vs bare, automated). 
**M1** — T1 ring buffer + zstd transcripts + replay endpoint; T2 state engine + pack loader + claude-code & codex packs; T3 fixture transcripts recorded into `fixtures/` + table tests (stream → expected state timeline); T4 Board with sessions_delta + StateBadge; T5 fallback heuristics pack; T6 SIGKILL/exit detection (waitpid) → exited within 5 s test.
**M2** — T1 approval parser + `approval` frames + ApprovalSheet with confidence gating; T2 stdin/resize frames + multi-viewer fanout test; T3 PWA (manifest, sw, install), MobileKeysBar; T4 Web Push (subscribe, VAPID, events: waiting_*, done, exited≠0; per-session mute; quiet hours); T5 auth (tokens, argon2id; localhost bypass) + write_audit events; T6 secret-redaction in notification previews (regex pack: AWS keys, `sk-…`, PEM headers); T7 Playwright mobile-viewport approval flow vs fakeagent. *Ships MVP.*
**M3** — T1 DiffView (`git diff <start_ref>` via `diff/gitdiff.go`, rename/binary handling); T2 opencode + pi packs; T3 `PACKS.md` authoring guide + pack lint command (`deck packs lint`); T4 transcript search (SQLite FTS over decompressed segments, background index).
**M4 (v1)** — T1 hub/satellite: satellite dials `wss://hub/ws-host` with host token, re-registers sessions, relays frames; host_id namespacing in UI; T2 buffering/reconnect (satellite queues events 10 min); T3 Claude Code deep adapter: hooks config writer (`deck integrate claude-code`) → hook POSTs to daemon → structured tool-call timeline in Info tab; T4 session launcher (templates table + POST /sessions {template} spawning `deck run` server-side). 

## 9. Test mapping

Pack table tests per fixture transcript version (breakage = update pack fixture, not code). Approval mis-parse release-blocker: corpus of 30 prompt variants + 10 adversarial (options in output text but not a real prompt) — assert confidence gating. Chaos: daemon restart marks orphans exited; WS disconnect/reconnect resumes from ring buffer. Bench: PTY latency, 50-session board load.
