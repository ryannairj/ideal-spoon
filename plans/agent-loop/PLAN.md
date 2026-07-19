# Loam — a minimal, auditable personal agent loop

**Category:** AI infra · **Difficulty:** Medium · **Platform:** Self-hosted daemon + chat surfaces

> Honest note (the idea list asked "too saturated?"): yes, generic agent loops are saturated — openclaw/nanoclaw clones abound. This plan is worth building only with a sharp angle. The angle chosen: **radical auditability and least-privilege** — a personal agent whose every capability, memory write, and message is inspectable and permission-scoped, aimed at people who *want* an always-on agent but don't trust a 50k-star framework running `eval` on their laptop. Small enough to read in an afternoon (< 3k LOC core), that being a stated, CI-enforced feature.

## 1. Vision

An always-on personal agent reachable via Telegram/CLI/web that can: remember things you tell it, run scheduled routines ("every morning: weather + calendar + top emails digest"), and use a small set of *explicitly granted* tools. Every action lands in an append-only audit log you can actually read. The anti-framework: no plugin soup, no 200 tools, no YOLO shell access.

## 2. Why it can win

- Trust is the unserved niche. Capability-scoped tools + readable core + audit trail = the agent you'd let near your email.
- "Read the whole core in an afternoon" is a real differentiator against sprawling frameworks — enforced by a CI LOC budget (core ≤ 3,000 lines, excluding tools).

## 3. Users & use cases

Privacy-conscious tinkerers, self-hosters.

1. Telegram: "remind me to renew the car rego next Friday 9am" → scheduled, confirmed.
2. Morning routine at 7:00: digest message assembled from granted tools (calendar read, weather, RSS).
3. "What did I tell you about Sam's allergy?" → memory retrieval with source turn linked.
4. `loam audit --since yesterday` → every model call, tool call, token cost.

## 4. MVP scope & non-goals

**MVP:** daemon; Telegram + CLI surfaces; agent loop with tool-calling; capability system; memory (notes + retrieval); scheduler (reminders + routines); audit log; 6 built-in tools: `memory`, `schedule`, `web_fetch` (readonly, domain-allowlist option), `rss`, `weather`, `shell` (disabled by default, per-command allowlist when enabled).
**Non-goals:** multi-user, marketplace/plugins, browser automation, email *sending* (read-only IMAP tool in v1.1), voice, autonomous goal pursuit (it acts on request + schedule only).

## 5. Tech stack

- **Core:** Python 3.12, `uv`; SQLite; provider-agnostic LLM client (Anthropic/OpenAI-compatible/Ollama); no agent framework — the loop is ~200 lines and that's the point.
- **Surfaces:** `python-telegram-bot`; CLI (`loam chat`); minimal FastAPI web UI (read-mostly: audit, memory, config).
- **Embeddings** for memory retrieval (provider or local `fastembed`).

## 6. Architecture

```
surfaces (telegram/cli/web) → message bus → agent loop
  loop: context assembly (persona + granted tools + relevant memories + recent turns)
        → model → tool calls? → capability check → execute → observe → repeat (max N)
        → reply
scheduler (APScheduler) → injects routine/reminder events into the same loop
everything → audit log (append-only JSONL + SQLite index)
```

**Capability system (the heart):** tools declare scopes (`memory:write`, `web:fetch:<domain>`, `shell:<binary>`), grants live in `grants.toml`, hot-reloaded; loop passes only granted tools to the model; execution re-checks grants (defense in depth); denials are surfaced to the user, not silently swallowed. High-risk scopes (shell, any future write-scope) additionally require per-invocation confirmation via the chat surface unless marked `trusted` in grants.

## 7. Data model

- `turns(id, surface, direction, content, at)` · `memories(id, text, source_turn_id, embedding, tags, created_at, superseded_by)`
- `schedules(id, kind{reminder,routine}, cron_or_at, payload_json, enabled, last_run_at)`
- `audit(id, at, kind{model_call,tool_call,tool_denied,confirm_request,config_change}, detail_json, tokens_in, tokens_out, cost)`
- `grants` from TOML (not DB) — config-as-file, versionable in the user's dotfiles.

## 8. Feature specs

**F1 — Loop + tools.** Max 8 tool iterations/turn; timeouts per tool; structured errors fed back to model once, then abort with honest failure message. *AC:* tool exception never crashes the daemon; audit row exists for every model/tool call (invariant test).

**F2 — Capabilities.** As §6. *AC:* revoking a grant in grants.toml takes effect next turn without restart; a model attempting an ungranted tool yields `tool_denied` audit + user-visible note; shell tool refuses anything outside its allowlist including via shell metacharacters (release-blocking injection test suite: `;`, `&&`, backticks, `$()`, pipes).

**F3 — Memory.** Explicit ("remember that…") + agent-initiated saves (always tagged as such); retrieval = hybrid embed+keyword, top-k into context with source links; supersede/edit/forget commands; memory browser in web UI. *AC:* "forget X" hard-deletes and audit-logs; retrieval eval (30 seeded memories/15 queries) ≥ 85% hit in top-3.

**F4 — Scheduler.** Reminders (one-shot, natural-language time parsing with confirm-echo) + routines (cron + prompt template with tool access). Missed-while-down policy: fire once on start if < 12 h late. *AC:* DST-crossing schedules fire correctly (fake-clock tests, tz from config).

**F5 — Surfaces.** Telegram: auth by chat-id allowlist; markdown replies; long-output truncation with "full in web UI" link. CLI: interactive + one-shot `loam ask "…"`. Web: audit browser, memory browser, grants viewer (read-only; edit stays in TOML), cost dashboard. *AC:* an unauthorized chat-id gets zero information (not even "unauthorized" details) and an audit row.

**F6 — Auditability.** JSONL log rotation; `loam audit` CLI with filters; monthly cost rollup; `loam verify` re-checks LOC budget + grants sanity. *AC:* replaying a day's JSONL reconstructs the SQLite index (round-trip test).

## 9. Milestones

- **M0 (1):** loop + CLI surface + anthropic/openai providers + audit JSONL. *Ships: readable core.*
- **M1 (2):** capability system + memory + the 4 safe tools; injection test suite.
- **M2 (3):** scheduler + Telegram + confirmations for risky scopes. *Ships: MVP daily-driver.*
- **M3 (4):** web UI (audit/memory/cost), shell tool behind allowlist, docs + `LOAM_TOUR.md` (the read-it-in-an-afternoon guide), LOC-budget CI. *Ships: v1.0.*

## 10. Testing

- Injection suite (F2) release-blocking; fuzz natural-language time parsing against confirm-echo correctness.
- Recorded-model fixtures for loop unit tests; live nightly smoke.
- 48 h soak test in CI-adjacent env: memory leak / crash-free / schedule drift checks.

## 11. Risks & open questions

- Saturation risk is real: if the trust angle stops being differentiating, fold the good parts (capability system, audit) into other plans (agent-deck, token-furnace) — noted exit criterion.
- NL time parsing errors → always confirm-echo parsed time in the reply; reminders are cheap to cancel.
