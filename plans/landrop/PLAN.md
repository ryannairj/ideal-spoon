# LANdrop — AirDrop for everything on your network, zero install

**Category:** Social / utility · **Difficulty:** Easy–Medium · **Platform:** Single binary host + browser clients (every device)

## 1. Vision

Getting a file from your Android phone to your Mac, or a link from your work laptop to your personal one, still routes through cloud chats and email-to-self. LANdrop is a tiny server you run on one machine (or your NAS): every device on the Wi-Fi opens `drop.local`, gets a name, sees every other connected device, and flings files, text snippets, and clipboard contents at them — instantly, LAN-only, no accounts, no cloud, no apps.

## 2. Why it can win

- Snapdrop/PairDrop exist but are P2P-WebRTC-only (flaky on corporate/guest Wi-Fi, nothing persists). The wedge: **a host with a drop-box** — transfers relay through the host when P2P fails (100% success on any network) and a shared "shelf" holds items for devices that aren't online yet.
- Zero install is the whole product: if grandma's iPad has a browser, it's a first-class citizen.

## 3. Users & use cases

Households, small offices, workshops/classrooms; anyone with 2+ devices.

1. Phone → open PWA → tap "Kitchen iPad" → send 30 photos → iPad notification, saves to camera roll via browser download.
2. Copy text on laptop A → "send clipboard" → laptop B taps to copy.
3. Put `invoice.pdf` on the **shelf** → partner grabs it tomorrow from their phone.
4. Guest mode: visitor scans the QR on your fridge, can send to you but can't browse the shelf.

## 4. MVP scope & non-goals

**MVP:** single binary host; device presence + naming (fun auto-names, editable); direct sends (files multi-select, folders as zip, text/links, clipboard) with accept/decline; WebRTC P2P with automatic host-relay fallback; the shelf (persistent shared space with expiry); PWA with share-target (Android) and web-share; QR join; guest mode; mDNS `drop.local`; transfer resume.
**Non-goals:** internet/WAN relay (LAN only — that's the privacy promise), accounts/identity beyond device names, E2E encryption in MVP (LAN + TLS; document threat model; E2E in v1.1), native apps, chat/messaging (text snippets are fire-and-forget, not threads).

## 5. Tech stack

- **Host:** Go 1.23 single binary: HTTP server (chi) + WebSocket signaling/presence + relay endpoints + SQLite (shelf metadata, device registry) + blob dir; mDNS advertise (`zeroconf`); self-signed TLS with on-screen fingerprint (or plain HTTP mode for simplicity — default HTTP on LAN with clear labeling, HTTPS optional since getUserMedia isn't needed; **WebRTC requires secure context on some browsers → decision: host serves HTTPS via self-signed + QR contains cert-pin bypass instructions; relay path works on HTTP regardless**).
- **Client:** React PWA served by host; WebRTC data channels (`simple-peer`-style thin wrapper); streams via File System Access API where available, else download.

## 6. Architecture

```
all devices ──WS (presence, signaling, transfer offers)──► host
sender ──WebRTC data channel──► receiver          (fast path)
sender ──chunked POST──► host ──SSE/stream──► receiver   (relay fallback, auto after 5s P2P failure)
shelf: sender POSTs to host storage; receivers list/download; expiry sweeper
```

Transfers are chunked (1 MB), hash-verified (BLAKE3 whole-file), resumable by chunk bitmap on both paths. Presence rooms keyed by network interface (multi-subnet hosts show segmented rooms).

## 7. Data model (host SQLite)

- `devices(id, name, emoji, ua_hint, first_seen, last_seen, is_guest)`
- `transfers(id, from_device, to_device, kind{files,text,clipboard}, manifest_json /*names, sizes, hashes*/, path{p2p,relay}, status{offered,accepted,active,done,declined,failed}, bytes_done, started_at, finished_at)`
- `shelf_items(id, device_id, kind, name, size, blob_ref, text_content, expires_at, downloads, pinned)`
- `settings(key, value)` — guest policy, shelf quota/expiry (default 7 d, 5 GB), name.

## 8. Feature specs

**F1 — Presence & identity.** Open page → device gets persisted identity (localStorage token) + auto-name ("Brave Mango"); grid of online devices with platform icons; rename/emoji. *AC:* two devices see each other < 2 s after page load; reconnect (sleep/wake) restores identity without duplicate entries.

**F2 — Send flow.** Multi-select files (or drag-drop / paste / share-target) → pick device → receiver gets accept dialog (names, total size, sender) → progress both sides → hash verify → receiver saves. Text/links render with copy button + open button; "clipboard" kind auto-copies on accept where API allows. *AC:* 2 GB file transfers successfully on both paths (P2P and forced-relay), survives a 10 s Wi-Fi drop mid-transfer (resume), and hash-verifies; 100-photo batch arrives as individual files (File System Access) or a zip fallback.

**F3 — Relay fallback.** Auto-switch after P2P setup timeout; visible path indicator ("direct"/"via host"); host relay streams without buffering whole files (constant memory). *AC:* relay of 2 GB uses < 100 MB host RSS; corporate-Wi-Fi simulation (client isolation container network) still transfers via relay.

**F4 — The shelf.** Shared space: anything droppable; items show uploader, size, countdown to expiry; pin to keep; quota with oldest-unpinned eviction warning. *AC:* expiry sweeper removes blob + row atomically (no orphans — GC test); guest devices can add to shelf but see only their own items (policy test).

**F5 — Join & guest mode.** Host prints/serves a QR (URL + optional room PIN); guest toggle per policy: guests send-to-named-devices only. *AC:* wrong-PIN devices see nothing; flipping guest policy applies without restart.

**F6 — PWA polish.** Installable; Android share-target ("Share → LANdrop"); Web Push optional for accept prompts when tab is backgrounded; dark mode; works down to iOS Safari 16. *AC:* share-target flow from Android gallery → device picker in < 3 taps; backgrounded receiver still gets the offer (push or on-focus queue).

## 9. Milestones

- **M0 (1):** host binary + presence + text sends + basic UI. *Ships: usable for links/snippets.*
- **M1 (2):** file transfers via relay path with chunking/resume/hash. *Ships: reliable core.*
- **M2 (3):** WebRTC fast path + auto-fallback + folder/batch handling. 
- **M3 (4):** shelf + expiry + quotas; QR join + guest mode. *Ships: MVP.*
- **M4 (5):** PWA share-target/push, mDNS + TLS story, multi-subnet rooms, docs (NAS/Docker/Pi guides). *Ships: v1.0.*

## 10. Testing

- Transfer torture: chunk-loss injection, mid-transfer disconnects, concurrent sends to one receiver, zero-byte files, 4 GB file, filename edge cases (unicode, reserved chars) — both paths.
- Multi-client E2E: Playwright with 3 browser contexts over the real host binary; forced-relay mode via WebRTC-disabled context.
- Host memory/leak soak under sustained relay load.

## 11. Risks & open questions

- Browser storage/save ergonomics vary (iOS can't write folders) → per-platform save strategy table maintained in docs; zip fallback everywhere.
- Secure-context requirements for WebRTC vs self-signed friction → relay path is the guaranteed floor on plain HTTP; treat P2P as progressive enhancement (this is the key architectural bet).
- mDNS blocked on some routers → QR/IP join always works; `drop.local` is convenience, not dependency.
