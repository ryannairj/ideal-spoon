# Termius Clone — cross-platform SSH client with synced hosts, snippets, and SFTP

**Category:** Clone / dev tool · **Difficulty:** Hard · **Platforms:** Desktop (macOS/Linux/Windows) first, mobile later

## 1. Vision

A beautiful, fast SSH client where your hosts, keys (encrypted), snippets, and settings follow you across devices. One app replaces `~/.ssh/config` spelunking, scattered terminal profiles, and "which IP was that box again?" Termius proved the market; we build the open, self-hostable version where *you* own the sync server.

## 2. Why it can win

- **Self-hosted sync** is the wedge. Termius sync is closed and subscription-gated; teams with compliance needs can't use it. Our sync server is a single binary you run yourself; E2E-encrypted so the server never sees secrets.
- Local-first: fully functional with sync disabled — no account wall.
- Import everything: `~/.ssh/config`, PuTTY sessions, Termius export.

## 3. Users & use cases

DevOps engineers, homelab owners, freelancers juggling client servers.

1. Connect to a saved host in ≤ 2 keystrokes (fuzzy launcher).
2. Organize hosts in groups with shared settings (jump host, port, identity) inherited down the tree.
3. Run a saved snippet ("restart nginx") against one or many hosts.
4. Browse files over SFTP with drag-drop upload/download alongside the terminal.
5. New laptop: log into sync, everything appears; vault decrypts with passphrase.

## 4. MVP scope & non-goals

**MVP (M0–M3):** desktop app with host management, terminal tabs/splits, key management, snippets, SFTP panel, `~/.ssh/config` import, local encrypted vault.
**v1 (M4–M5):** self-hosted sync server + E2E encryption, multi-exec snippets, port-forwarding UI.

**Non-goals:** mobile apps (post-v1), Mosh/telnet/serial protocols, team RBAC/sharing, terminal AI features.

## 5. Tech stack

- **Shell:** Tauri 2 (Rust core, small binaries) with React + TypeScript UI.
- **Terminal rendering:** `xterm.js` + WebGL addon.
- **SSH engine:** Rust `russh` (SSH + SFTP) in the Tauri core; terminal I/O streamed to the webview over Tauri channels. *Do not* shell out to system `ssh` (breaks Windows parity and programmatic control).
- **Local store:** SQLite via `rusqlite`; secrets encrypted with `age`-style X25519 + passphrase-derived key (argon2id), never stored plaintext.
- **Sync server:** Rust (axum) single binary + SQLite; clients push/pull encrypted blobs only.

## 6. Architecture

```
┌─ Tauri webview (React) ─────────────────┐
│ HostList │ Terminal(xterm.js) │ SFTP UI │
└────────────┬────────────────────────────┘
       Tauri IPC / channels
┌────────────┴────────────────────────────┐
│ Rust core                               │
│  session mgr ── russh (SSH/SFTP)        │
│  vault (SQLCipher-like enc layer)       │
│  sync client (push/pull enc blobs) ─────┼──► sync server (axum+SQLite)
└─────────────────────────────────────────┘
```

- One Rust `SessionManager` owns all live SSH connections; each terminal tab maps to a channel. UI is stateless w.r.t. connections (survives webview reload).
- Sync model: every entity has `uuid`, `updated_at`, `deleted` (tombstone). Last-writer-wins per field group. Server stores opaque encrypted rows — key never leaves clients.

## 7. Data model (SQLite, local)

- `groups(id, parent_id, name, sort)` — settings inherit down the tree.
- `hosts(id, group_id, label, hostname, port, username, identity_id, jump_host_id, tags_json, color, notes, last_connected_at)`
- `identities(id, label, kind{key,password,agent}, private_key_enc, public_key, passphrase_enc)`
- `snippets(id, label, script, confirm_before_run)`
- `known_hosts(host_id, key_type, fingerprint, approved_at)`
- `settings(key, value_json)` — theme, font, shell prefs.
- `sync_state(entity, uuid, version, synced_at)`

## 8. Feature specs

**F1 — Host management.** CRUD hosts/groups; group tree in left sidebar; effective settings = host overrides ⊕ ancestor group defaults. *AC:* creating a host under a group with a jump host set connects through that jump host with zero host-level config.

**F2 — Terminal.** Tabs + horizontal/vertical splits; scrollback ≥ 10k lines; search; copy-on-select option; per-host color/label on tab. *AC:* `vim`, `htop`, `tmux` render correctly; resize propagates to remote PTY; disconnect shows a reconnect banner, and reconnect restores the tab.

**F3 — Fuzzy launcher.** `Cmd/Ctrl+K` opens a palette matching label/hostname/tag; Enter connects. *AC:* palette open→connected in under 150 ms of app-side latency on a warm host entry.

**F4 — Identities & agent.** Import/generate ed25519 keys; use OS agent when available; per-host identity selection. *AC:* private keys are unreadable in the DB file (verified by test that greps raw DB for key material).

**F5 — Known-host trust.** First connect shows fingerprint prompt; mismatch shows a hard warning (no "ignore" default). *AC:* changed server key blocks connection until the user explicitly re-approves.

**F6 — Snippets.** CRUD; run in current tab; v1: run against N selected hosts with per-host result list. *AC:* snippet run against 3 hosts shows 3 statuses independently (ok/fail + output).

**F7 — SFTP panel.** Toggleable file pane per host: list, upload/download (drag-drop), rename, delete, chmod, and transfer queue with progress. *AC:* 500 MB upload survives a UI navigation; queue shows throughput and completes.

**F8 — Import.** Parse `~/.ssh/config` (Host, HostName, User, Port, IdentityFile, ProxyJump) into hosts/groups; PuTTY registry/session import on Windows. *AC:* a 50-host ssh config round-trips into working host entries.

**F9 — E2E sync (v1).** Opt-in account on self-hosted server; entities serialized→encrypted client-side (XChaCha20-Poly1305, key from passphrase via argon2id)→pushed. Conflict = LWW with conflict copy kept. *AC:* server DB contains no plaintext hostnames (test greps server DB); two clients converge after offline edits to the same host.

## 9. Milestones

- **M0 – Skeleton (1)**: Tauri app boots; xterm.js tab talks to a local shell PTY; CI builds macOS/Linux/Windows artifacts.
- **M1 – SSH core (2)**: russh connect with key/password; host CRUD + groups + inheritance; known-host prompts; fuzzy launcher. *Ships as: usable basic client.*
- **M2 – Daily driver (3)**: splits, search, scrollback, reconnect; identities + agent; ssh-config import; settings/themes. *Ships as: can replace terminal for SSH work.*
- **M3 – SFTP + snippets (4)**: F6, F7. *Ships as: MVP release v0.1.*
- **M4 – Sync server (5)**: axum server, accounts (email+password → argon2id), encrypted blob push/pull, device management, Docker image.
- **M5 – Sync client + multi-exec (6)**: client sync engine, conflict handling, snippet multi-exec, port-forwarding UI (L/R/D forwards with status). *Ships as: v1.0.*

## 10. Testing

- Integration tests run against `openssh-server` in Docker (password + key + jump-host topologies via docker-compose).
- Property test for group-inheritance resolution.
- Sync: simulated two-client divergence suite; fuzz the blob decrypt path.
- E2E (Playwright driving Tauri via WebDriver): connect → run command → assert output; SFTP upload/download round-trip checksum.

## 11. Risks & open questions

- **russh PTY/feature gaps** (agent forwarding, some KEX): budget M1 time for validation; fallback is `libssh2` bindings, not shelling out.
- **Windows key storage**: use DPAPI-wrapped vault key on Windows, Keychain on macOS, Secret Service on Linux; passphrase fallback everywhere.
- Termius import format is undocumented — best-effort, feature-flagged.
