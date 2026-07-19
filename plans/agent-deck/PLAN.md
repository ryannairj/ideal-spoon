# Agent Deck — mission control for AI CLI agents, from any device

**Category:** Dev tool · **Difficulty:** Hard · **Platforms:** Local daemon + web UI (desktop & mobile PWA)

## 1. Vision

You've got Claude Code in three repos, a Codex run upstairs, an OpenCode session from yesterday — each trapped in its own terminal. Agent Deck is the single pane of glass: every agent session across every machine, live, in one dashboard you can open on your phone. See who's working, who's stuck waiting for permission, approve/deny, send follow-ups, review diffs — from the couch.

## 2. Why it can win

- The multi-agent workflow is exploding and the tooling is per-vendor. Nobody owns the **cross-agent** view.
- Key insight: most CLI agents are just **PTY programs with recognizable states**. A generic PTY-wrapper adapter gets 80% of value for *any* agent tool immediately; deeper protocol adapters (where APIs exist, e.g. Claude Code hooks/headless mode) layer on structured events.
- Mobile approval flow ("Agent wants to run `rm -rf dist` — Allow?") is a killer feature no terminal gives you.

## 3. Users & use cases

Power users running 2+ coding agents; small teams with a shared dev box.

1. Launch: `deck run -- claude` (or codex/opencode/pi/anything) inside any repo → session appears on the dashboard.
2. Glance at phone: 4 sessions — 2 working, 1 waiting for approval, 1 done. Tap the waiting one, read the tool request, approve.
3. Send "also add tests for the edge cases" to a running session from the browser.
4. Review the session's diff (`git diff` of its worktree) before telling it to commit.
5. Get a push notification when any session finishes or errors.

## 4. MVP scope & non-goals

**MVP:** single-machine daemon + web UI; generic PTY adapter with state detection for Claude Code, Codex CLI, OpenCode; live terminal view (read + write); status board; diff viewer; PWA + push notifications; token auth.
**v1:** multi-machine (agents on N hosts → one hub), Claude Code deep adapter (hooks → structured tool-call feed, headless `-p` runs), session templates/queues.
**Non-goals:** being an agent itself (no LLM calls), vendor cloud sync, Windows daemon in MVP (WSL is fine), native mobile apps (PWA only).

## 5. Tech stack

- **Daemon (`deckd`):** Go 1.23 — PTY management (`creack/pty`), SQLite state, WebSocket + REST server. Single binary.
- **UI:** React + TypeScript + Vite PWA; `xterm.js` for terminal panes; served by daemon.
- **Push:** Web Push (VAPID) — works on installed PWAs on iOS ≥ 16.4 and Android.
- **Multi-machine (v1):** satellite daemons dial the hub over WebSocket (outbound-only, token-authed); hub relays.

## 6. Architecture

```
deck run -- <agent cmd>          web / phone PWA
        │                              │
        ▼                              ▼
   deckd (Go) ── PTY per session ── WS/REST ── UI
        ├─ ring buffer (scrollback 50k lines/session)
        ├─ state engine (per-agent regex/heuristic packs)
        ├─ SQLite (sessions, events, transcripts)
        └─ notifier (web push)
```

**State engine:** each adapter pack defines detectors over the PTY stream + process signals → session state machine: `starting → working → waiting_input → waiting_approval → idle/done → exited(code)`. Detectors are data (JSON regex/timeout rules per tool + version), hot-reloadable — new agent CLI = new pack, no rebuild. Fallback heuristics when no pack matches: output-silence + prompt-pattern → `waiting_input`.

## 7. Data model

- `sessions(id, host_id, cwd, repo_root, git_branch, cmd, adapter, state, title, started_at, ended_at, exit_code, pinned)`
- `events(id, session_id, at, kind{state_change,approval_request,notice,user_input}, payload_json)`
- `transcripts(session_id, seq, chunk BLOB)` — compressed PTY output for replay.
- `hosts(id, name, token_hash, last_seen)` (v1)
- `push_subscriptions(id, endpoint, keys_json, filters_json)`
- `auth_tokens(id, name, token_hash, created_at)`

## 8. Feature specs

**F1 — Session capture.** `deck run -- <cmd>` wraps any command in a PTY, registers with daemon, streams output; `deck attach` for an already-running tmux-style takeover is out of scope (document workaround: launch via deck). *AC:* wrapped Claude Code behaves identically to bare (interactive keys, colors, resize); overhead < 5 ms added latency.

**F2 — Status board.** Cards per session: agent icon, repo/branch, state with duration ("waiting for approval · 6 min"), last meaningful output line, cost line if the agent prints one. Sort: needs-attention first. *AC:* state transitions reflect on the board < 1 s; a session killed with SIGKILL shows `exited` within 5 s.

**F3 — Live terminal.** Full xterm.js view per session, read/write, scrollback, mobile-usable keyboard incl. Esc/Ctrl/arrow bar. *AC:* two browsers viewing one session both see output and either can type; replay of a finished session renders the full transcript.

**F4 — Approval fast-path.** When state = `waiting_approval`, extract the pending question/tool request (adapter pack captures the prompt text and option list) and render as buttons (y/n/option numbers → keystrokes to PTY). *AC:* Claude Code permission prompt is answerable with one tap from the phone; mis-parse falls back to raw terminal, never sends wrong keys silently.

**F5 — Notifications.** Web push on: enters `waiting_approval`/`waiting_input`, `done`, `exited != 0`; per-session mute; quiet hours. *AC:* notification delivered to installed iOS PWA in test; tapping opens that session.

**F6 — Diff view.** For sessions inside a git repo: working-tree diff vs session-start ref, file list + unified diff, mobile-friendly. *AC:* diff matches `git diff` output; binary files listed not rendered.

**F7 — Security.** Localhost by default; `--listen` requires token auth (argon2id-hashed tokens) + TLS or explicit `--insecure`; all writes audit-logged to `events`. *AC:* unauthenticated WS/REST rejected; secret scan of transcripts before push notification previews (redact obvious keys).

**F8 — Multi-host (v1).** `deckd --satellite wss://hub token=…`; sessions namespaced by host; hub outage → satellites buffer events and reconnect. *AC:* hub restart loses no session states after satellites reconnect.

## 9. Milestones

- **M0 (1):** daemon + `deck run` PTY wrapping + raw web terminal for one session. 
- **M1 (2):** session registry, status board, state engine + packs for Claude Code & Codex, SQLite persistence/replay. *Ships: single-machine dashboard.*
- **M2 (3):** approval fast-path, write access, mobile PWA polish + web push, auth. *Ships: MVP — the phone-approval demo.*
- **M3 (4):** diff viewer, OpenCode + pi packs, adapter-pack docs so the community can add tools, transcript search.
- **M4 (5–6):** multi-host hub/satellite, Claude Code hooks deep adapter (structured tool-call timeline), session launcher (start new agent runs from UI with saved templates). *Ships: v1.0.*

## 10. Testing

- **Fake agent** binary in repo: scriptable CLI that emits recorded output patterns of each supported tool (from fixture transcripts) — deterministic adapter tests without API keys.
- State-engine table tests: fixture stream → expected state timeline, per pack per version.
- Chaos tests: daemon restart mid-session (PTY children must survive via reparenting or be reported dead honestly — decide: sessions die with daemon in MVP, documented), WS disconnect/reconnect.
- E2E: Playwright mobile-viewport approval flow against fake agent.

## 11. Risks & open questions

- **Agent CLIs change their prompts** → packs are versioned data with a fixture-transcript CI suite; breakage = update pack, not code. Fallback heuristics keep sessions usable meanwhile.
- Approval mis-parse is the scariest failure (tapping "allow" sends wrong keys) → strict parse-confidence threshold; below it, buttons disabled, raw terminal only. Release-blocking tests.
- iOS PWA push has platform quirks — test early in M2; ntfy.sh fallback channel as escape hatch.
