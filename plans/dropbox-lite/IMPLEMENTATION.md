# Implementation Spec — Dropbox-Lite ("Dropling")

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Names | server `droplingd`, client `dropling`, module `github.com/OWNER/dropling` |
| License | MIT · Language Go 1.23 for both binaries |
| Server | chi v5, SQLite WAL via `sqlc`-generated queries, listen `:8080`; blob dir `$DATA_DIR/blobs/ab/cd/<sha256>` |
| Client store | SQLite at `~/.config/dropling/state.db`; watcher `fsnotify` + 2 s debounce; full rescan on start + every 6 h |
| Transfer | resumable chunked upload, 4 MiB chunks, offset protocol (tus-inspired, own routes); 4 parallel transfers |
| Hashing | SHA-256 whole file (streamed); rename detection by hash |
| Auth | argon2id passwords; web session cookie (30 d); device tokens `dl_<32 bytes base62>` stored hashed |
| Change feed | long-poll `GET /v1/changes?cursor=&timeout=30s` (SSE deliberately not used — simpler client) |
| Conflict copy name | `"{stem} (conflict from {host} {YYYY-MM-DD}){ext}"` |
| Web UI | React 18 + TS + Vite, Tailwind, TanStack Query; embedded |
| TLS | `--behind-proxy` (default) or `--autocert domain.com` |
| Tray | build tag `tray` using `fyne.io/systray`; CLI-first |

## 1. Repository layout

```
dropling/
  cmd/droplingd/main.go           # serve, admin-reset, gc
  cmd/dropling/main.go            # login, sync (daemon), status, pause, resume, logout
  internal/server/
    api/{routes.go, auth.go, nodes.go, blobs.go, changes.go, shares.go, trash.go, versions.go, admin.go}
    store/{schema.sql, queries.sql, sqlc-generated…}
    blob/{store.go, gc.go}        # CAS write (tmp+rename), refcount ops in same tx as node ops
    journal/journal.go            # append + cursor reads (THE core)
    zipstream/zipstream.go        # folder download, constant memory
  internal/client/
    sync/{engine.go, scanner.go, watcher.go, pull.go, push.go, conflicts.go}
    state/{schema.sql, store.go}
    api/client.go                 # typed HTTP client, retries w/ backoff
    ignore/ignore.go              # .droplingignore (gitignore syntax via go-gitignore)
    tray/tray.go
  internal/proto/PROTOCOL.md      # normative sync protocol doc (written in M1, kept current)
  internal/conformance/           # M1 harness: drives 2 fake clients against server
    harness.go scenarios/*.go
  web/src/pages/{Files.tsx, Preview.tsx, Trash.tsx, Versions.tsx, Shares.tsx, ShareView.tsx,
                 Admin.tsx, Login.tsx}  components/{Uploader.tsx, Breadcrumbs.tsx, NodeRow.tsx}
  .github/workflows/ci.yml        # incl. case-sensitivity matrix job (mac + linux)
  Dockerfile  docker-compose.yml
```

## 2. Dependencies

Go: `go-chi/chi/v5`, `modernc.org/sqlite`, `sqlc` (codegen, committed output), `fsnotify/fsnotify`, `alexedwards/argon2id`, `golang.org/x/crypto/acme/autocert`, `rs/zerolog`, `spf13/cobra` (CLIs), `sabhiram/go-gitignore`. Web: react, tanstack-query, react-router, tailwind, `react-pdf` NOT used (PDF preview via native `<iframe>`), `@uppy/core`+`@uppy/dashboard` NOT used (own Uploader, keep it lean). Dev: vitest, playwright.

## 3. Configuration

Server env: `DROPLING_DATA_DIR` (default `/var/lib/dropling`), `DROPLING_LISTEN` (`:8080`), `DROPLING_BEHIND_PROXY` (bool), `DROPLING_AUTOCERT_DOMAIN`, `DROPLING_TRASH_DAYS` (30), `DROPLING_VERSIONS_KEEP` (10), `DROPLING_DEFAULT_QUOTA_GB` (0 = unlimited). First boot prints one-time admin setup URL. Client config file `~/.config/dropling/config.json` `{server, token, syncRoot, selectedPaths[], excludePaths[]}`.

## 4. Server schema (sqlc `schema.sql`)

```sql
CREATE TABLE users (id TEXT PRIMARY KEY, email TEXT UNIQUE NOT NULL, pw_hash TEXT NOT NULL,
  is_admin INTEGER NOT NULL DEFAULT 0, quota_bytes INTEGER NOT NULL DEFAULT 0, created_at INTEGER NOT NULL);
CREATE TABLE devices (id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id),
  name TEXT NOT NULL, token_hash TEXT NOT NULL, last_seen_at INTEGER);
CREATE TABLE nodes (id TEXT PRIMARY KEY, user_id TEXT NOT NULL REFERENCES users(id),
  parent_id TEXT REFERENCES nodes(id), name TEXT NOT NULL,
  kind TEXT NOT NULL CHECK(kind IN ('file','dir')),
  sha256 TEXT, size INTEGER NOT NULL DEFAULT 0, mtime INTEGER,
  version INTEGER NOT NULL DEFAULT 1, deleted_at INTEGER);
CREATE UNIQUE INDEX nodes_unique_name ON nodes(user_id, parent_id, name) WHERE deleted_at IS NULL;
CREATE TABLE journal (seq INTEGER PRIMARY KEY AUTOINCREMENT, user_id TEXT NOT NULL,
  node_id TEXT NOT NULL, op TEXT NOT NULL CHECK(op IN ('upsert','delete','move')),
  snapshot_json TEXT NOT NULL, at INTEGER NOT NULL);
CREATE INDEX journal_user ON journal(user_id, seq);
CREATE TABLE versions (node_id TEXT NOT NULL, version INTEGER NOT NULL, sha256 TEXT NOT NULL,
  size INTEGER NOT NULL, mtime INTEGER, at INTEGER NOT NULL, PRIMARY KEY(node_id, version));
CREATE TABLE blobs (sha256 TEXT PRIMARY KEY, size INTEGER NOT NULL, refcount INTEGER NOT NULL DEFAULT 0);
CREATE TABLE uploads (id TEXT PRIMARY KEY, user_id TEXT NOT NULL, sha256 TEXT NOT NULL,
  size INTEGER NOT NULL, received INTEGER NOT NULL DEFAULT 0, tmp_path TEXT NOT NULL, created_at INTEGER);
CREATE TABLE shares (id TEXT PRIMARY KEY, token TEXT UNIQUE NOT NULL, node_id TEXT NOT NULL,
  user_id TEXT NOT NULL, password_hash TEXT, expires_at INTEGER, allow_upload INTEGER DEFAULT 0, created_at INTEGER);
```

Client `state.db`: `local_nodes(path PRIMARY KEY, node_id, sha256, size, mtime, base_version, state CHECK(state IN ('synced','dirty','pulling','pushing','conflict')))`, `pending_ops(seq, kind, path, detail)`, `meta(key, value)` (cursor, device id).

## 5. API contract (`/v1`, bearer device-token or session cookie)

| Endpoint | Semantics |
|---|---|
| `POST /auth/register` (admin-only after first user) / `POST /auth/login` `{email,pw,deviceName}` → `{token, deviceId}` | |
| `GET /nodes/{id}` · `GET /nodes/{id}/children` · `GET /nodes/path?p=/a/b` | metadata reads |
| `POST /nodes` `{parentId, name, kind, sha256?, size?, mtime?, baseVersion?}` | create/commit-file; **412** if node exists with version ≠ baseVersion (conflict signal) |
| `PATCH /nodes/{id}` `{name?, parentId?, baseVersion}` | move/rename; 412 on version mismatch |
| `DELETE /nodes/{id}` | → trash (tombstone) |
| `HEAD /blobs/{sha256}` | 200 exists / 404 (dedupe check) |
| `POST /uploads` `{sha256, size}` → `{uploadId, received}` · `PATCH /uploads/{id}` (raw bytes, `Upload-Offset` header) · `POST /uploads/{id}/finish` | resumable upload; finish verifies hash, moves to CAS |
| `GET /blobs/{sha256}` | download (supports `Range`) |
| `GET /changes?cursor=N&timeout=30` | `{entries:[{seq, op, node:{…snapshot}}], cursor}`; long-poll |
| `GET /nodes/{id}/versions` · `POST /nodes/{id}/restore` `{version}` | versioning |
| `GET /trash` · `POST /trash/{id}/restore` | trash |
| `GET /nodes/{id}/zip` | streamed zip of folder |
| Shares: `POST /shares`, `GET /shares`, `DELETE /shares/{id}`; public: `GET /s/{token}` (page), `GET /s/{token}/download`, `POST /s/{token}/upload` (file-request mode). Expired/bad password → 404 always. |
| `GET /admin/users`… admin CRUD · `GET /healthz` | |

## 6. Sync engine (client) — the normative algorithm

```
loop:
  PULL: changes since cursor → for each entry (ordered by seq):
    if local path unchanged since base_version → apply (write file via tmp+rename / mkdir / move / delete→local trash dir)
    if local dirty at same path → CONFLICT: keep local as conflict copy (rename local), then apply remote
    advance cursor after each applied entry (crash-safe)
  PUSH: for each pending_op (fifo):
    file change → hash → HEAD blob → upload if missing → POST/PATCH node with baseVersion
      on 412 → refetch remote node → write conflict copy from local → re-push conflict copy as new node
    move detected (same hash vanished at A, appeared at B within one scan) → PATCH parent/name (no upload)
  WATCH: fsnotify events → debounce 2 s per path → enqueue scan of subtree
Scanner compares (size, mtime) first; hashes only when changed. Ignore: .droplingignore + always-ignore list (.DS_Store, Thumbs.db, ~$*, *.tmp, .dropling-tmp/*).
```

Case handling: server is case-sensitive-unique per §4 index but rejects (`409 name_case_collision`) creating a name differing only by case from an existing sibling; client on case-insensitive FS renames incoming collisions to `"{name} (case conflict)"` — documented in PROTOCOL.md.

## 7. Web UI screens

`Files` (breadcrumbs, list w/ virtualized rows, drag-drop + folder upload via webkitdirectory, context menu: rename/move/delete/share/versions, zip-download for dirs) · `Preview` (image/pdf/text/video by content-type, else download card) · `Trash`, `Versions` (list + restore), `Shares` (manage), `ShareView` (public, minimal, password gate), `Admin` (users, quotas), `Login`.

## 8. Milestone task lists

**M0** — T1 scaffold + sqlc setup + migrations; T2 auth (register/login/devices) + admin bootstrap; T3 nodes CRUD + journal writes on every mutation; T4 CAS blob store + resumable uploads + Range downloads; T5 CI + Docker. *Done when curl can do a full file lifecycle.*
**M1** — T1 `/changes` long-poll + cursor pagination (500/page); T2 PROTOCOL.md written (request/response examples for every §5 route + §6 algorithm); T3 conformance harness: fake-client lib + scenarios (fresh sync, offline edits both sides, rename vs edit, delete vs edit, case-only rename, rapid writes, clock skew) — runs against real server in-process; T4 412/409 semantics + tests. *Done when all scenarios pass with a scripted reference client.*
**M2** — T1 client skeleton (login/config/state.db); T2 scanner + hasher + pending_ops; T3 pull applier (tmp+rename, crash-safe cursor); T4 push + conflict copies + move detection; T5 watcher + debounce; T6 `status`/`pause`/`resume` (unix socket control); T7 run conformance scenarios against the real client (replaces fake client). *Done when 2 real clients converge on the torture suite.*
**M3** — T1 web Files + Uploader + Preview; T2 trash + restore; T3 zipstream + folder download; T4 shares (CRUD, public page, password, expiry, file-request upload); T5 Playwright suite. *Ships v0.1.*
**M4** — T1 selective sync (selectedPaths in config; excluded subtrees tracked in state as `remote-only`, confirm-dialog on exclude); T2 versions (server keep-N + restore + UI); T3 blob GC (`droplingd gc`: refcount==0 AND not in trash-window → delete; property test with random op sequences); T4 quotas enforcement (upload rejects over-quota, 413); T5 tray build + autocert; T6 admin UI; tag v1.0.

## 9. Test mapping

Conformance scenarios are the release gate (`internal/conformance`, run in CI on every PR; each scenario asserts convergence + zero-loss invariant: every byte-version ever acked by server remains retrievable via versions/trash within retention). Case-sensitivity CI job runs client tests on macOS runner (case-insensitive) + linux. Load test script (`test/load/`): 100k nodes journal pull stays O(changes) — assert query plans use `journal_user` index.
