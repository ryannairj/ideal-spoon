# Implementation Spec — Termius Clone

Read `PLAN.md` first for scope and acceptance criteria. This file makes the implementation decisions. Do not re-decide anything here; log unavoidable deviations in `DECISIONS.md`.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Product/binary name | **Moor** (`moor` desktop app, `moord` sync server) |
| License | MIT |
| Desktop shell | Tauri 2.x, React 18, TypeScript 5, Vite 6 |
| Rust toolchain | stable, edition 2021, workspace at repo root |
| SSH library | `russh` 0.45+ / `russh-sftp` (fallback decision pre-made: if agent-forwarding blocks M2, switch to `ssh2` crate for the agent path only) |
| Local DB | SQLite via `rusqlite` (bundled), file `moor.db` in app data dir |
| Secret crypto | XChaCha20-Poly1305 (`chacha20poly1305` crate), key = argon2id(passphrase, salt), params m=64MiB t=3 p=4 |
| OS keychain | `keyring` crate stores the vault key wrapped; passphrase fallback always available |
| Sync server | Rust, `axum` 0.8, SQLite via `sqlx`, listens `:8977` |
| UI state | Zustand; styling: Tailwind CSS 4; terminal: `@xterm/xterm` 5 + `@xterm/addon-webgl`, `addon-fit`, `addon-search` |
| IPC | Tauri commands (request/response) + Tauri channels (PTY streams) |
| ID scheme | UUIDv7 strings everywhere |

## 1. Repository layout

```
moor/
  Cargo.toml                    # workspace: crates/*, src-tauri
  package.json                  # pnpm workspace: apps/desktop
  crates/
    moor-core/                  # all non-UI logic; no Tauri deps
      src/
        lib.rs
        vault.rs                # encryption, key derivation, keychain wrap
        db.rs                   # rusqlite pool, migrations runner
        migrations/0001_init.sql …
        models.rs               # Host, Group, Identity, Snippet, KnownHost structs
        settings.rs
        session/
          mod.rs                # SessionManager: HashMap<SessionId, Session>
          ssh.rs                # russh connect, auth ladder, PTY channel
          sftp.rs               # sftp ops + transfer queue
          jump.rs               # ProxyJump chain builder
        import/
          ssh_config.rs         # ~/.ssh/config parser
          putty.rs              # windows registry sessions
        sync/
          client.rs             # push/pull encrypted blobs
          merge.rs              # LWW merge, tombstones, conflict copies
          wire.rs               # blob format v1 (serde + encrypt)
    moor-sync-server/           # moord binary
      src/{main.rs, routes.rs, auth.rs, store.rs}
      migrations/0001_init.sql
  src-tauri/
    src/main.rs                 # command registration only; thin wrappers over moor-core
    src/commands/{hosts.rs, sessions.rs, sftp.rs, snippets.rs, vault.rs, sync.rs, import.rs}
    tauri.conf.json
  apps/desktop/                 # React app loaded by Tauri
    src/
      main.tsx  App.tsx  router.tsx
      stores/{hosts.ts, sessions.ts, ui.ts, settings.ts}
      lib/ipc.ts                # typed wrappers for every Tauri command (single source of IPC truth)
      components/
        layout/{Sidebar.tsx, TabBar.tsx, StatusBar.tsx}
        hosts/{HostTree.tsx, HostForm.tsx, GroupForm.tsx, EffectiveSettings.tsx}
        terminal/{TerminalPane.tsx, SplitLayout.tsx, ReconnectBanner.tsx}
        launcher/CommandPalette.tsx
        sftp/{SftpPanel.tsx, TransferQueue.tsx, FileRow.tsx}
        snippets/{SnippetList.tsx, SnippetRunModal.tsx, MultiExecResults.tsx}
        identity/{IdentityList.tsx, KeyGenModal.tsx}
        trust/FingerprintPrompt.tsx
        settings/SettingsPage.tsx
        vault/{UnlockScreen.tsx, VaultSetup.tsx}
  tests/
    docker/docker-compose.yml   # openssh-server ×3: password, key, jumphost topology
    integration/                # rust integration tests hitting the containers
  .github/workflows/ci.yml      # lint+test, tauri build matrix (mac/linux/win)
```

## 2. Dependencies (exact)

Rust (`moor-core`): `russh`, `russh-sftp`, `russh-keys`, `rusqlite {bundled}`, `chacha20poly1305`, `argon2`, `keyring`, `uuid {v7}`, `serde`, `serde_json`, `thiserror`, `tokio {rt-multi-thread}`, `tracing`. Server adds: `axum`, `sqlx {sqlite,runtime-tokio}`, `tower-http {trace,limit}`, `argon2`, `rand`.
Frontend: `react`, `react-dom`, `zustand`, `@xterm/xterm` + addons above, `@tauri-apps/api`, `fuse.js` (palette fuzzy), `tailwindcss`, `clsx`. Dev: `vitest`, `@testing-library/react`, `typescript`, `eslint`, `prettier`.

## 3. Configuration

Desktop: no env vars; all settings in `settings` table. App data dir via `tauri::api::path::app_data_dir`.
Server env: `MOOR_DB_PATH` (default `./moord.db`), `MOOR_LISTEN` (default `0.0.0.0:8977`), `MOOR_REGISTRATION` (`open|invite|closed`, default `open`). Ship `.env.example` + Dockerfile.

## 4. Database schema (local, `crates/moor-core/src/migrations/0001_init.sql`)

```sql
CREATE TABLE groups (
  id TEXT PRIMARY KEY, parent_id TEXT REFERENCES groups(id) ON DELETE CASCADE,
  name TEXT NOT NULL, sort INTEGER NOT NULL DEFAULT 0,
  default_username TEXT, default_port INTEGER, default_identity_id TEXT,
  default_jump_host_id TEXT, updated_at INTEGER NOT NULL, deleted INTEGER NOT NULL DEFAULT 0);
CREATE TABLE hosts (
  id TEXT PRIMARY KEY, group_id TEXT REFERENCES groups(id) ON DELETE SET NULL,
  label TEXT NOT NULL, hostname TEXT NOT NULL, port INTEGER, username TEXT,
  identity_id TEXT REFERENCES identities(id), jump_host_id TEXT REFERENCES hosts(id),
  tags TEXT NOT NULL DEFAULT '[]', color TEXT, notes TEXT,
  last_connected_at INTEGER, updated_at INTEGER NOT NULL, deleted INTEGER NOT NULL DEFAULT 0);
CREATE TABLE identities (
  id TEXT PRIMARY KEY, label TEXT NOT NULL,
  kind TEXT NOT NULL CHECK(kind IN ('key','password','agent')),
  private_key_enc BLOB, public_key TEXT, passphrase_enc BLOB, password_enc BLOB,
  updated_at INTEGER NOT NULL, deleted INTEGER NOT NULL DEFAULT 0);
CREATE TABLE snippets (
  id TEXT PRIMARY KEY, label TEXT NOT NULL, script TEXT NOT NULL,
  confirm_before_run INTEGER NOT NULL DEFAULT 0,
  updated_at INTEGER NOT NULL, deleted INTEGER NOT NULL DEFAULT 0);
CREATE TABLE known_hosts (
  id TEXT PRIMARY KEY, host_pattern TEXT NOT NULL, key_type TEXT NOT NULL,
  fingerprint_sha256 TEXT NOT NULL, approved_at INTEGER NOT NULL,
  UNIQUE(host_pattern, key_type));
CREATE TABLE settings (key TEXT PRIMARY KEY, value_json TEXT NOT NULL);
CREATE TABLE sync_state (
  entity TEXT NOT NULL, entity_id TEXT NOT NULL, version INTEGER NOT NULL DEFAULT 0,
  synced_at INTEGER, PRIMARY KEY(entity, entity_id));
```

Server DB: `users(id, email UNIQUE, pw_hash, created_at)`, `devices(id, user_id, name, token_hash, last_seen_at)`, `blobs(user_id, entity, entity_id, version, ciphertext BLOB, updated_at, deleted, PRIMARY KEY(user_id, entity, entity_id))`.

## 5. IPC contract (Tauri commands — implement exactly these; `lib/ipc.ts` mirrors them)

| Command | Args | Returns |
|---|---|---|
| `vault_status` | – | `{state: 'uninitialized'|'locked'|'unlocked'}` |
| `vault_setup` / `vault_unlock` / `vault_lock` | `{passphrase}` / `{passphrase}` / – | `Result<()>` |
| `hosts_list` / `groups_list` | – | full trees incl. deleted=0 only |
| `host_upsert` / `host_delete` / `group_upsert` / `group_delete` | entity JSON / `{id}` | entity / `()` |
| `host_effective_settings` | `{hostId}` | resolved `{username, port, identityId, jumpHostId}` + provenance per field |
| `session_open` | `{hostId, cols, rows}` | `{sessionId}`; emits channel `pty://{sessionId}` (bytes) and event `session_state://{sessionId}` (`connecting|trust_prompt|connected|disconnected|closed|error{msg}`) |
| `session_write` / `session_resize` / `session_close` | `{sessionId, dataB64}` / `{sessionId, cols, rows}` / `{sessionId}` | `()` |
| `trust_respond` | `{sessionId, accept: bool}` | `()` |
| `identity_list/upsert/delete/generate_ed25519/import_file` | … | … |
| `snippet_list/upsert/delete` · `snippet_run` | `{snippetId, sessionId}` | `()` (types into PTY) |
| `snippet_multi_exec` | `{snippetId, hostIds[]}` | `{execId}`; events `exec://{execId}` per-host `{hostId, status, output}` |
| `sftp_open/list/mkdir/rename/delete/chmod` | `{sessionId, path…}` | listings/`()` |
| `sftp_transfer_start` | `{sessionId, direction, localPath, remotePath}` | `{transferId}`; events `transfer://{transferId}` `{bytes, total, state}` |
| `sftp_transfer_cancel` | `{transferId}` | `()` |
| `import_ssh_config` | `{path?}` | `{preview: HostDraft[]}` then `import_commit {drafts}` |
| `sync_configure/status/now` | `{serverUrl, email, password}` / – / – | status incl. last sync, pending count |

Server HTTP API (`moord`): `POST /v1/register`, `POST /v1/login` → `{deviceToken}`, `GET /v1/blobs?since_version=` (paged), `PUT /v1/blobs` (batch upsert, returns per-item new version), `DELETE /v1/devices/{id}`. Auth: `Authorization: Bearer <deviceToken>`. All payload ciphertext is opaque to the server.

## 6. UI screens

Single window. Left sidebar (host tree + search), center tab strip (terminal tabs, split via `SplitLayout` — binary tree of panes, max depth 3), right slide-over SFTP panel, bottom status bar (latency, sync state, vault lock). Modals: HostForm, KeyGen, FingerprintPrompt, SnippetRun, CommandPalette (Cmd/Ctrl+K), VaultSetup/Unlock (blocking screen). Keyboard map: Cmd/Ctrl+K palette, Cmd/Ctrl+T new tab (palette-pick host), Cmd/Ctrl+W close tab, Cmd/Ctrl+D split-right, Cmd/Ctrl+Shift+D split-down, Cmd/Ctrl+F terminal search.

## 7. Core algorithms

**Effective settings** (`host_effective_settings`): walk host → group → ancestors; first non-null wins per field; provenance = which node supplied it. Cycle-guard on jump chains (max depth 4, error `JumpChainTooDeep`).

**Auth ladder** (ssh.rs): try in order — explicit identity (key w/ passphrase → prompt via event if needed), OS agent (if identity.kind=agent or none set), password (prompt event). Each failure logged to session state; 3 password attempts max.

**Sync merge** (merge.rs): pull server blobs since local cursor → decrypt → for each entity: if local `updated_at` > remote → keep local, queue push; if remote > local → overwrite local; if both changed since last sync (version mismatch on push, HTTP 409) → keep remote, clone local as `"<label> (conflict <device> <date>)"` new entity. Tombstones (`deleted=1`) replicate like edits; purge after 90 days.

**Jump-host depth limit (why 4).** The cycle-guard cap of 4 is a safety bound, not a protocol limit: it terminates accidental cycles and pathological configs while covering every realistic topology (client → bastion → inner-bastion → target = depth 3; 4 leaves one hop of headroom). Deeper chains are almost always a misconfiguration; erroring with `JumpChainTooDeep` is safer than opening N nested connections. Configurable later via settings if a real need appears; hard-coded for MVP.

**russh fallback trigger (concrete criteria).** The `ssh2`-crate fallback for the agent path is taken only if, during M2 T4, **any** of these is observed against the docker `agent` topology: (a) `russh` agent-forwarding fails to authenticate against a standard `ssh-agent`/`gpg-agent` socket after the auth ladder completes (reproducible in integration test `ac_agent_forward.rs`), or (b) forwarded-agent channel open returns unsupported/`ChannelOpenFailure` on ≥2 of the 3 test servers. "Blocking" = the agent AC cannot pass with `russh` alone. Scope of fallback: agent auth path only; password/key paths stay on `russh`. Decision + evidence logged in `DECISIONS.md`.

## 7a. Sync conflict-resolution UI

When `merge.rs` creates a conflict copy, the UI surfaces it non-destructively:

- The cloned entity `"<label> (conflict <device> <date>)"` appears in the host tree with a **conflict badge** (amber dot) next to both it and the surviving original.
- Clicking the badge opens a **Conflict drawer**: side-by-side field diff (local-kept vs conflict-copy), per-field "use this value" buttons, and a single **Resolve** action that writes the chosen merge into the original and soft-deletes (`deleted=1`) the conflict copy.
- No auto-merge of field values ever happens silently — LWW decides which row *survives as canonical*, but the user always sees that a divergence occurred until they dismiss/resolve it.
- Status bar shows a conflict count; "Resolve all later" is allowed (copies simply remain as normal hosts, fully usable).

## 8. Milestone task lists

**M0** — T1 scaffold workspace exactly as §1 (`pnpm create tauri-app` then restructure); T2 CI (fmt+clippy+vitest+`tauri build` matrix); T3 local PTY tab: `portable-pty` dev-only shell wired through `session_open` path to prove channel plumbing; T4 xterm pane + fit/webgl addons. *Done when CI produces 3 artifacts and typing in the local shell works.*

**M1** — T1 migrations + models + db.rs; T2 vault.rs + setup/unlock screens (blocking); T3 russh connect (password+key) replacing dev PTY; T4 known-hosts flow (`trust_prompt` state + FingerprintPrompt); T5 host/group CRUD UI + effective settings + provenance chips; T6 jump-host chain; T7 CommandPalette (fuse.js over label/hostname/tags, Enter = `session_open`); T8 docker test topology + integration tests (connect via password, key, jump). *Done when PLAN F1/F3/F5 ACs pass.*

**M2** — T1 SplitLayout + tab persistence (reopen last session's tabs' hosts); T2 reconnect banner + retry (keeps tab, new session id); T3 scrollback 10k + search addon; T4 identities UI, ed25519 keygen (russh-keys), agent auth, per-host identity; T5 ssh_config importer (support: Host, HostName, User, Port, IdentityFile, ProxyJump; globs become groups) + preview/commit UI; T6 settings page (font, theme, copy-on-select, scrollback size); T7 DB-greps-clean test for key material.

**M3** — T1 sftp.rs ops + panel UI; T2 transfer queue (concurrent 3, chunked read/write 256KiB, progress events, cancel, resume-on-reopen NOT required); T3 drag-drop upload + download-to picker; T4 snippets CRUD + run-in-tab; T5 tag v0.1, package installers (dmg/AppImage/msi via tauri bundler).

**M4** — T1 moord: schema, register/login (argon2id), device tokens, blob endpoints, Docker image; T2 rate limiting (tower-http, 10 rps/device) + `/healthz`; T3 server integration tests incl. "ciphertext only" grep test.

**M5** — T1 wire.rs blob format `{v:1, entity, id, nonce, ct}`; T2 sync client (push queue on every upsert, pull on interval 60 s + manual); T3 merge.rs + conflict copies + tests (two simulated clients, §10 scenarios); T4 snippet multi-exec (bounded parallelism 5, per-host session, capture last 4KiB output); T5 port-forward UI (L/R/D forms → russh channels, status list with stop). *Done when PLAN F6/F9 ACs pass; tag v1.0.*

## 9. Test mapping

Every PLAN acceptance criterion gets a test with matching name: `crates/moor-core/tests/ac_f5_known_host_mismatch.rs` style for Rust; `apps/desktop/src/**/*.ac.test.tsx` for UI-level via vitest; docker-based integration suite runs in CI on linux only (mac/win build-only).
