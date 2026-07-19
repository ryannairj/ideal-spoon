# FolderSync Clone — rule-based folder sync/backup engine with a friendly scheduler UI

**Category:** Clone / utility · **Difficulty:** Medium · **Platforms:** Desktop daemon + web UI (Android later)

## 1. Vision

"Keep *this* folder synced to *that* place, on *this* schedule, with *these* rules" — as a set-and-forget appliance. FolderSync Pro nails this on Android; desktop users are stuck stitching together rclone cron jobs. We wrap a proven transfer engine (rclone) in a first-class **folderpair + scheduler + history** experience.

## 2. Why it can win

- rclone already supports 70+ storage backends but is expert-only. The product is the *management layer*: folderpairs, schedules, filters, conflict policy, run history, notifications — not the transfer protocol.
- Local web UI means one codebase for desktop/NAS/headless server; perfect for homelab.

## 3. Users & use cases

Homelab/NAS owners, photographers, small offices.

1. Mirror `~/Pictures` → Backblaze B2 nightly at 02:00, excluding RAWs > 1 GB.
2. Two-way sync `~/Documents` ↔ Google Drive every 15 min with conflict copies.
3. One-way ingest from an SFTP camera-upload folder into local NAS, delete source after transfer.
4. See exactly what changed in last night's run, and get a push/email on failure.

## 4. MVP scope & non-goals

**MVP:** daemon + web UI; folderpairs with one-way mirror & two-way sync; schedules; include/exclude filters; run history with per-file log; local↔local, S3-compatible, SFTP, WebDAV, Google Drive, Dropbox backends (whatever rclone config supports, but these six are tested).
**Non-goals:** mobile apps, real-time watch mode (v1.1 via fsnotify debounce), block-level dedup/versioned backup (that's restic's job — link, don't build), multi-user.

## 5. Tech stack

- **Daemon:** Go 1.23; embeds the sync engine by driving **rclone as a library** (`librclone` RC API) — battle-tested backends and bisync for free.
- **Store:** SQLite (`modernc.org/sqlite`), migrations via `goose`.
- **Scheduler:** `robfig/cron/v3` inside the daemon.
- **Web UI:** React + TypeScript + Vite, served embedded (`embed.FS`); REST + SSE for live progress.
- **Packaging:** single static binary; Docker image; systemd unit + launchd plist generators (`app install-service`).

## 6. Architecture

```
web UI (React) ──REST/SSE──► daemon (Go)
                              ├─ scheduler (cron)
                              ├─ run orchestrator (queue, 1 run per pair)
                              ├─ librclone (backends, sync/bisync/move)
                              └─ SQLite (pairs, remotes, runs, file events)
```

- Orchestrator serializes runs per folderpair, allows N pairs in parallel (configurable, default 2).
- Each run: dry-run pass (optional) → transfer → parse rclone JSON logs into `file_events` rows → finalize run status → fire notifications.
- Remote credentials stored in rclone config format, encrypted at rest (age key stored in OS keychain or `--config-key` for headless).

## 7. Data model

- `remotes(id, name, type, rclone_config_enc, created_at)`
- `folderpairs(id, name, src_remote_id, src_path, dst_remote_id, dst_path, mode{mirror,two_way,move}, filters_json, conflict_policy{newer_wins,keep_both,src_wins}, delete_policy{never,mirror,after_transfer}, bandwidth_limit, enabled)`
- `schedules(id, pair_id, cron_expr, only_on_ac_power, only_on_networks_json)`
- `runs(id, pair_id, trigger{cron,manual,api}, started_at, finished_at, status{ok,partial,failed,cancelled}, files_transferred, bytes, deletes, errors, log_path)`
- `file_events(run_id, path, action{copy,update,delete,conflict,skip,error}, size, detail)`
- `notifications(id, kind{webhook,email,ntfy}, config_json, on{failure,always})`

## 8. Feature specs

**F1 — Remote management.** Add/test remotes via guided forms per backend (not raw rclone config); OAuth flows for Drive/Dropbox open browser and capture token. *AC:* "Test" button validates credentials by listing the remote root; failures show the backend error verbatim.

**F2 — Folderpair CRUD + modes.** Mirror (src→dst incl. deletes), two-way (rclone bisync with `--conflict-resolve` mapped from conflict_policy), move (delete source after verify). *AC:* two-way conflict with `keep_both` produces `file (conflict YYYY-MM-DD).ext` and logs a `conflict` event.

**F3 — Filters.** Include/exclude glob lists, min/max size, min age; UI shows a live "what would match" preview against the source tree. *AC:* preview and actual run agree on a fixture tree of 1k files.

**F4 — Scheduling.** Cron editor with human-readable summary and next-3-runs preview; constraints (Wi-Fi SSID allowlist — best effort per OS, AC power). *AC:* misfired schedules (daemon asleep) run once on wake if within grace window (configurable, default 1 h).

**F5 — Runs & history.** Live progress (files/bytes/percent, current file) over SSE; per-run file event table, filterable by action; retention pruning (default 90 days). *AC:* cancelling a run stops transfers within 5 s and marks status `cancelled`; partial transfers are not corrupted (rclone temp-file semantics preserved).

**F6 — Safety rails.** Pre-run check refuses a mirror run that would delete > X% of destination (default 25%, per-pair override) — the classic "empty source nukes backup" guard. *AC:* automated test: emptying source dir causes run to abort with `failed: delete guard`, destination untouched.

**F7 — Notifications.** ntfy/webhook/email (SMTP) on failure or always; message includes pair, status, counts, top errors. *AC:* failure notification fires exactly once per run.

**F8 — Headless & API.** Everything in the UI is available over the authenticated REST API (`X-API-Key`); OpenAPI spec published at `/api/openapi.json`. *AC:* a pair can be created, run, and its history fetched purely via API in the E2E suite.

## 9. Milestones

- **M0 (1):** Go daemon skeleton + embedded UI + SQLite migrations + CI (lint, test, cross-compile matrix).
- **M1 (2):** remotes (local, SFTP, S3) + folderpair CRUD + manual mirror runs with live progress. *Ships: useful one-way sync tool.*
- **M2 (3):** scheduler, filters + preview, run history + file events, delete guard. *Ships: set-and-forget backups.*
- **M3 (4):** two-way sync via bisync, conflict policies, move mode, Drive/Dropbox/WebDAV OAuth backends. *Ships: MVP v0.1.*
- **M4 (5):** notifications, retention, service installers, Docker/NAS docs, API keys + OpenAPI. *Ships: v1.0.*

## 10. Testing

- Backend matrix in CI: local↔local, MinIO (S3), `atmoz/sftp` container. Fixture trees with unicode names, symlinks, 0-byte files, deep nesting.
- bisync conflict suite: edit-edit, edit-delete, create-create collisions per policy.
- Delete-guard and cancellation are release-blocking tests.
- Playwright E2E: create remote → pair → run → verify history renders events.

## 11. Risks & open questions

- **bisync maturity**: rclone bisync has documented edge cases; keep two-way behind a "beta" label until the conflict suite is green across 3 consecutive rclone versions.
- Wi-Fi SSID detection is OS-specific and flaky — ship as best-effort, document.
- Google Drive OAuth verification for a published app takes time; ship with "bring your own client ID" documented.
