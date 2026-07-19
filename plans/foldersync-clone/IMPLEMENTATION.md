# Implementation Spec — FolderSync Clone

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Product/binary name | **Syncwell** (`syncwell` single binary) |
| License | MIT |
| Language | Go 1.23, module `github.com/OWNER/syncwell` |
| Sync engine | rclone embedded via `librclone` RC API (`github.com/rclone/rclone/librclone`) — never shell out |
| DB | SQLite (`modernc.org/sqlite`), file `$DATA_DIR/syncwell.db`, migrations via `pressly/goose` embedded |
| HTTP | `chi` v5; UI served from `embed.FS`; listen `:8384` |
| Scheduler | `robfig/cron/v3` with seconds disabled, tz from org setting |
| UI | React 18 + TS + Vite, Tailwind 4, TanStack Query, `dist/` embedded |
| Auth | single admin password (argon2id) → session cookie; plus `X-API-Key` rows |
| Remote config crypto | `filippo.io/age` X25519; key wrapped in OS keychain (`zalando/go-keyring`) or `SYNCWELL_CONFIG_KEY` env for headless |
| Concurrency | max 2 concurrent pair runs (setting `max_parallel_runs`), 1 run per pair enforced by per-pair mutex |
| Delete guard default | abort mirror if deletes > 25% of dst files AND > 10 files |

## 1. Repository layout

```
syncwell/
  cmd/syncwell/main.go            # flags: serve (default), install-service, version
  internal/
    server/{server.go, routes.go, auth.go, sse.go, apikeys.go}
    store/{db.go, migrations/*.sql, queries.go}   # hand-written queries, no ORM
    remotes/{remotes.go, forms.go, oauth.go}      # backend form defs + rclone config mgmt
    pairs/pairs.go
    engine/
      orchestrator.go             # run queue, per-pair serialization
      run.go                      # run lifecycle: guard → transfer → parse → finalize
      rclone.go                   # librclone wrapper: sync/copy/bisync/move, JSON log tap
      guard.go                    # delete-guard pre-check (dry-run stats)
      filters.go                  # filters_json → rclone filter rules + preview
    sched/scheduler.go            # cron + misfire grace + constraints
    notify/{notify.go, ntfy.go, webhook.go, smtp.go}
    service/{systemd.go, launchd.go, windows.go}
  web/                            # React app; `pnpm build` → embedded
    src/pages/{Dashboard.tsx, Remotes.tsx, RemoteForm.tsx, Pairs.tsx, PairForm.tsx,
               PairDetail.tsx, RunDetail.tsx, Settings.tsx, Login.tsx}
    src/components/{RunProgress.tsx, FilterPreview.tsx, CronEditor.tsx, EventTable.tsx}
    src/lib/{api.ts, sse.ts, types.ts}
  test/
    fixtures/tree1k/              # generator script committed, tree gitignored
    integration/                  # compose: minio, atmoz/sftp
  .github/workflows/ci.yml        # lint, unit, integration (linux), cross-compile matrix
  Dockerfile
```

## 2. Dependencies

Go: `github.com/rclone/rclone` (librclone + backends: local, sftp, s3, webdav, drive, dropbox — build tags to trim the rest), `modernc.org/sqlite`, `pressly/goose/v3`, `go-chi/chi/v5`, `robfig/cron/v3`, `filippo.io/age`, `zalando/go-keyring`, `alexedwards/argon2id`, `rs/zerolog`.
Web: `react`, `@tanstack/react-query`, `react-router-dom`, `tailwindcss`, `cron-parser` (next-runs preview), `zod`. Dev: `vitest`, `playwright`.

## 3. Configuration

Env: `SYNCWELL_DATA_DIR` (default `~/.local/share/syncwell`), `SYNCWELL_LISTEN` (`127.0.0.1:8384`), `SYNCWELL_CONFIG_KEY` (headless age key), `SYNCWELL_LOG_LEVEL`. First run with no admin password → setup page. `.env.example` + compose file provided.

## 4. Database schema

```sql
CREATE TABLE remotes (id TEXT PRIMARY KEY, name TEXT UNIQUE NOT NULL, type TEXT NOT NULL,
  rclone_config_enc BLOB NOT NULL, created_at INTEGER NOT NULL);
CREATE TABLE folderpairs (id TEXT PRIMARY KEY, name TEXT NOT NULL,
  src_remote_id TEXT NOT NULL REFERENCES remotes(id), src_path TEXT NOT NULL,
  dst_remote_id TEXT NOT NULL REFERENCES remotes(id), dst_path TEXT NOT NULL,
  mode TEXT NOT NULL CHECK(mode IN ('mirror','two_way','move')),
  filters_json TEXT NOT NULL DEFAULT '{}',
  conflict_policy TEXT NOT NULL DEFAULT 'keep_both' CHECK(conflict_policy IN ('newer_wins','keep_both','src_wins')),
  delete_guard_pct INTEGER NOT NULL DEFAULT 25, bandwidth_limit TEXT,
  enabled INTEGER NOT NULL DEFAULT 1, created_at INTEGER NOT NULL);
CREATE TABLE schedules (id TEXT PRIMARY KEY, pair_id TEXT NOT NULL REFERENCES folderpairs(id) ON DELETE CASCADE,
  cron_expr TEXT NOT NULL, only_on_ac INTEGER NOT NULL DEFAULT 0, networks_json TEXT NOT NULL DEFAULT '[]',
  grace_minutes INTEGER NOT NULL DEFAULT 60);
CREATE TABLE runs (id TEXT PRIMARY KEY, pair_id TEXT NOT NULL REFERENCES folderpairs(id),
  trigger TEXT NOT NULL CHECK(trigger IN ('cron','manual','api')),
  started_at INTEGER NOT NULL, finished_at INTEGER,
  status TEXT NOT NULL CHECK(status IN ('running','ok','partial','failed','cancelled')),
  files_transferred INTEGER DEFAULT 0, bytes INTEGER DEFAULT 0, deletes INTEGER DEFAULT 0,
  errors INTEGER DEFAULT 0, error_summary TEXT, log_path TEXT);
CREATE INDEX runs_pair ON runs(pair_id, started_at DESC);
CREATE TABLE file_events (run_id TEXT NOT NULL REFERENCES runs(id) ON DELETE CASCADE,
  seq INTEGER NOT NULL, path TEXT NOT NULL,
  action TEXT NOT NULL CHECK(action IN ('copy','update','delete','conflict','skip','error')),
  size INTEGER, detail TEXT, PRIMARY KEY(run_id, seq));
CREATE TABLE notifications (id TEXT PRIMARY KEY, kind TEXT NOT NULL CHECK(kind IN ('webhook','email','ntfy')),
  config_json TEXT NOT NULL, on_event TEXT NOT NULL CHECK(on_event IN ('failure','always')), enabled INTEGER DEFAULT 1);
CREATE TABLE api_keys (id TEXT PRIMARY KEY, name TEXT, key_hash TEXT NOT NULL, created_at INTEGER);
CREATE TABLE settings (key TEXT PRIMARY KEY, value_json TEXT NOT NULL);
```

## 5. API contract (`/api/v1`, JSON; cookie or `X-API-Key`)

| Method+Path | Body → Response |
|---|---|
| `POST /auth/login` | `{password}` → cookie |
| `GET/POST /remotes`, `PUT/DELETE /remotes/{id}` | remote CRUD; config fields per-form, encrypted at rest |
| `POST /remotes/{id}/test` | – → `{ok, listing?: string[], error?}` (lists root, 10 s timeout) |
| `POST /remotes/oauth/start` | `{type}` → `{authUrl, state}`; `GET /remotes/oauth/callback` completes |
| `GET/POST /pairs`, `PUT/DELETE /pairs/{id}` | pair CRUD (validates remotes exist, src≠dst) |
| `POST /pairs/{id}/preview-filters` | – → `{matched: string[], total, truncatedAt: 500}` |
| `POST /pairs/{id}/run` | `{dryRun?: bool}` → `{runId}` (409 if already running) |
| `GET /pairs/{id}/runs?limit&before` | run list |
| `GET /runs/{id}` / `GET /runs/{id}/events?action=&q=&page=` | run + paged events |
| `POST /runs/{id}/cancel` | → 202 |
| `GET /events/stream` | SSE: `run_progress {runId, pct, bytesPerSec, currentFile}`, `run_done {runId, status}` |
| `GET/POST /schedules`… | CRUD; `GET /schedules/{id}/next?n=3` preview |
| `GET/POST /notify`… + `POST /notify/{id}/test` | CRUD + test-fire |
| `GET /api/openapi.json` | generated spec (hand-maintained YAML source in repo) |
| `GET /healthz` | `{ok, version}` |

## 6. UI screens

`Dashboard` (pair cards: last status chip, next run, run-now button, live progress bar via SSE) · `Remotes` + `RemoteForm` (per-type field forms; S3: endpoint/key/secret/bucket; SFTP: host/port/user/key-or-pass; OAuth types: connect button) · `PairForm` (3-step: endpoints → mode/policies (radio cards with plain-English explanations + delete-guard slider) → filters with live preview) · `PairDetail` (runs table) · `RunDetail` (summary header, filterable EventTable, cancel) · `Settings` (admin password, API keys, notifications, parallelism, service install snippets).

## 7. Core algorithms

**Run lifecycle (`run.go`):** acquire pair mutex → build rclone filter args → if mode=mirror: dry-run stats pass → `guard.go` check (`deletes > max(10, pct×dstCount)` → fail `delete guard`) → execute (`sync/copy`, `bisync` for two_way with `--conflict-resolve` mapped: newer_wins→`newer`, src_wins→`path1`, keep_both→`none` + suffix), `--use-json-log --stats 1s` tapped via librclone callbacks → stream parse: stats lines → SSE progress; per-file lines → `file_events` batch inserts (500/tx) → finalize status (`errors>0 && transferred>0` → `partial`) → notify.

**bisync first-run rule:** a new two_way pair's first run always executes `bisync --resync` (documented in UI as "baseline sync"); subsequent runs plain. Store `resync_done` flag in pair filters_json meta.

**Misfire grace (`scheduler.go`):** on startup, for each schedule compute `prev` fire time; if `now - prev < grace_minutes` and no run exists after `prev` → enqueue run with trigger `cron`.

**Cancellation:** context cancel into librclone job (`job/stop`), wait ≤ 5 s, mark `cancelled`.

## 8. Milestone task lists

**M0** — T1 repo scaffold per §1 + goose migrations; T2 chi server + embedded UI shell + login/setup; T3 CI (golangci-lint, go test, `GOOS` matrix build, web build); T4 Dockerfile.
**M1** — T1 remotes CRUD + age encryption + keychain/env key; T2 forms for local/sftp/s3 + test endpoint; T3 librclone wrapper + mirror run happy path; T4 run lifecycle + file_events parsing + RunDetail; T5 SSE progress; T6 integration compose (minio, sftp) + fixture tree generator. *Done when PLAN F1/F5-basic ACs pass.*
**M2** — T1 scheduler + grace + next-runs API; T2 filters (include/exclude globs, min/max size, min age) → rclone args + preview endpoint + FilterPreview UI; T3 delete guard + release-blocking test (empty source aborts); T4 retention pruning job (runs>90 d); T5 cancellation.
**M3** — T1 bisync two_way + conflict policy mapping + resync rule; T2 conflict `file_events` extraction; T3 move mode (copy+verify+delete-src); T4 OAuth flow (drive, dropbox; BYO client-id fields) + webdav; T5 bisync conflict test suite (edit-edit, edit-delete, create-create × 3 policies).
**M4** — T1 notifications (ntfy/webhook/smtp) + once-per-run dedupe + test-fire; T2 API keys + OpenAPI file + headless E2E test (create→run→history via curl); T3 `install-service` generators; T4 docs (NAS/Docker/headless key mgmt), tag v1.0.

## 9. Test mapping

AC-named tests: `internal/engine/guard_ac_test.go` (delete guard), `bisync_policies_ac_test.go`, `scheduler_grace_ac_test.go` (fake clock via injected `nowFn`), preview-vs-run agreement test on fixture tree; Playwright: remote→pair→run→history flow. Integration suite runs 3 rclone versions in a nightly matrix (bisync stability watch per PLAN §11).
