# Dropbox-Lite — self-hosted, lightweight file sync you can run on a potato

**Category:** Clone / self-hosted · **Difficulty:** Hard · **Platforms:** Server (single binary) + desktop sync client + web UI

## 1. Vision

The 20% of Dropbox that matters — a magic folder that syncs across machines, plus web access and share links — with none of the weight. One static binary + one data directory on the server. Nextcloud is a PHP aircraft carrier; Syncthing has no server-canonical copy or web/share story. This sits exactly between.

## 2. Why it can win

- **Deploy in 60 seconds**: `./dropling serve --data /srv/files` and you're done. SQLite, no external services.
- **Server-canonical model** (unlike Syncthing): the server holds truth, so web UI, share links, and partial sync are natural.
- Sync client is a small Go binary/tray app, not an Electron monster.

## 3. Users & use cases

Homelabbers, privacy-focused individuals, small teams (≤ 10 users).

1. Edit a file on laptop; it appears on desktop and web within seconds.
2. Send someone a share link (optionally password/expiry) to a file or folder.
3. Restore a file deleted yesterday (trash) or an earlier version (simple versioning).
4. Selective sync: keep `videos/` server-only on the small laptop.

## 4. MVP scope & non-goals

**MVP:** server (accounts, file store, delta sync API, web UI, share links, trash), desktop CLI+tray client for macOS/Linux/Windows with two-way sync and selective sync. Versioning: keep last N versions (default 10).
**Non-goals:** E2E encryption (document as future; TLS + at-rest is the story), real-time collaborative editing, mobile apps (web UI is responsive; camera upload later), federation, block-level dedup (whole-file content addressing only), conflict-free merging (conflict copies instead).

## 5. Tech stack

- **Server:** Go 1.23, `chi` router, SQLite (WAL) via `sqlc`; content-addressed blob store on disk (`blobs/ab/cd/<sha256>`); embedded React web UI.
- **Client:** Go; `fsnotify` for watch, same `sqlc` local DB for state; tray via `fyne systray` (CLI-first, tray optional).
- **Protocol:** HTTPS REST + long-poll/SSE for change notifications. Content-defined chunking is **out**; files are stored whole, uploaded with resumable chunked transfer (tus-style offsets).
- **Auth:** session cookies (web) + device tokens (clients); argon2id password hashes.

## 6. Architecture & sync protocol

```
client A ──REST/SSE──► server (Go, SQLite + blob store) ◄──REST/SSE── client B
                        └── web UI, share links
```

- Every user has a file tree of `nodes`; every change bumps a per-user monotonically increasing `cursor` (journal sequence).
- **Pull:** client sends last cursor → server returns ordered journal entries since (upsert/delete of nodes with metadata + content hash). Client applies to disk.
- **Push:** client detects local change → uploads blob if hash unknown (`HEAD /blobs/:sha`) → commits metadata op with parent cursor. If the node changed server-side since the client's base version → server rejects → client writes conflict copy `name (conflict from <host> <date>).ext` and re-pushes.
- Moves/renames are first-class journal ops (no re-upload). Deletes go to trash (tombstone, blob retained until trash purge + no other reference).

## 7. Data model (server SQLite)

- `users(id, email, pw_hash, quota_bytes, created_at)`
- `devices(id, user_id, name, token_hash, last_seen_at)`
- `nodes(id, user_id, parent_id, name, kind{file,dir}, sha256, size, mtime, version, deleted_at)` — unique `(user_id, parent_id, name)` where not deleted.
- `journal(seq, user_id, node_id, op{upsert,delete,move}, snapshot_json, at)` — the sync backbone.
- `versions(node_id, version, sha256, size, mtime, at)` — last N retained.
- `blobs(sha256, size, refcount)`
- `shares(id, token, node_id, user_id, password_hash, expires_at, allow_upload, created_at)`

Client DB mirrors: `local_nodes(path, sha256, size, mtime, base_version, state{synced,dirty,conflict})`, `config(server, token, selected_paths)`.

## 8. Feature specs

**F1 — Server & accounts.** Single-binary serve; first-run creates admin; admin page for users/quotas. *AC:* fresh VM → running server with TLS (built-in Let's Encrypt via `autocert` or `--behind-proxy`) in ≤ 3 commands.

**F2 — Delta sync.** As per protocol above. *AC:* 10k-file tree initial sync completes; touching 1 file causes exactly 1 blob upload + 1 journal op; edit-edit race produces a conflict copy and both contents survive (no data loss — release-blocking test).

**F3 — Efficient transfers.** Resumable uploads (offset-based), parallelism 4, hash-skip for unchanged/renamed content. *AC:* killing the client mid-500 MB-upload and restarting resumes, not restarts (verified by server-side byte counters).

**F4 — Selective sync.** Client `include`/`exclude` path prefixes; excluded subtrees tracked but not materialized. *AC:* excluding `videos/` frees local disk after confirmation; re-including restores files.

**F5 — Web UI.** Browse, upload (drag-drop, folder upload), download (folder → zip stream), preview images/PDF/text/video, move/rename/delete, trash restore, version restore. *AC:* Playwright suite covers each; folder zip of 2 GB streams without server OOM (constant memory).

**F6 — Share links.** Per node: token URL, optional password + expiry; folder shares get a minimal listing page; optional "file request" upload-only mode. *AC:* expired/wrong-password links return 404-indistinguishable responses.

**F7 — Trash & versions.** Trash retains 30 days (configurable); versions per F-spec. *AC:* restore of version k byte-identical to original (checksum test); blob GC never deletes a referenced blob (property test on refcount with random op sequences).

**F8 — Client UX.** `dropling login/sync/status/pause`, daemon mode with tray icon (sync state, recent changes, pause), `.droplingignore` patterns. *AC:* status shows per-file pending queue; ignore patterns prevent upload of matched files.

## 9. Milestones

- **M0 (1):** server skeleton, auth, node CRUD via REST, blob store, CI + Docker image.
- **M1 (2):** journal/cursor sync protocol server-side + protocol doc (`PROTOCOL.md`) + conformance test harness (drives two fake clients).
- **M2 (3–4):** Go client: scanner, watcher, push/pull loop, conflicts, resume. *Ships: CLI two-way sync between 2 machines.*
- **M3 (5):** web UI (browse/upload/preview/trash) + share links. *Ships: MVP v0.1.*
- **M4 (6):** selective sync, versions + restore, tray app, quotas/admin, GC job, autocert. *Ships: v1.0.*

## 10. Testing

- **Sync torture suite** (release-blocking): scripted scenarios — offline edits both sides, rename vs edit, delete vs edit, case-only renames, rapid successive writes, clock skew. Assert convergence + zero silent data loss.
- Property-based test: random op sequences on 2 simulated clients must converge with server.
- Filesystem quirks matrix: run client tests on case-insensitive (macOS/Windows) and case-sensitive FS in CI.
- Load: 100k nodes, journal pull pagination stays O(changes).

## 11. Risks & open questions

- **Correctness is the product** — a sync tool that loses data once is dead. Hence conformance harness in M1 *before* the real client, and the torture suite gating every release.
- Windows path oddities (reserved names, long paths): normalize and refuse-with-rename strategy, documented.
- Whole-file storage makes huge-file edits costly — acceptable for MVP; note rsync-delta as future work in `DECISIONS.md`.
