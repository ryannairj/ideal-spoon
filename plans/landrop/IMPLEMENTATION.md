# Implementation Spec — LANdrop

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **LANdrop** (`landrop` binary) · License MIT · Go 1.23 host + React PWA |
| Port | 5311 (`--port`); mDNS advertises `drop.local` + `_landrop._tcp` |
| TLS decision (final) | default plain HTTP on LAN; relay path is the guaranteed floor. `--tls self` generates self-signed (for browsers requiring secure context for WebRTC); QR encodes whichever scheme is active. P2P is progressive enhancement, auto-attempted only when `isSecureContext` |
| Chunking | 1 MiB chunks; whole-file BLAKE3 verify; resume via chunk bitmap (both paths) |
| P2P | WebRTC DataChannel, vanilla `RTCPeerConnection` (no lib) wrapped in `ui/src/lib/rtc.ts`; STUN: none (LAN — host candidates only); P2P setup timeout 5 s → relay |
| Relay | sender `POST /api/relay/{transferId}/chunks/{n}` → receiver streams `GET /api/relay/{transferId}/stream` (chunked response); constant-memory pipe via bounded channel (8 chunks buffered); backpressure = 429 + retry-after |
| Identity | device token localStorage `ld_device` (nanoid 21), server assigns auto-name from wordlists `adjective-fruit` ("Brave Mango") |
| Rooms | per server-interface subnet: room key = interface network (segmentation algorithm in §8); `--single-room` flag collapses |
| Shelf | default 7 d expiry, 5 GiB quota, oldest-unpinned eviction; blob dir `$DATA/shelf/` |
| Guest policy | `open` (default) \| `pin` (room PIN) \| `guests-limited` (send-only to non-guests, own-shelf-items-only) |
| Save strategy | File System Access API when available (directory batch), else per-file download; >10 files without FSA → server-zipped batch download |
| Push | Web Push VAPID optional (`--push`); offer prompt also queued for on-focus |
| Text/clipboard kinds | plain text ≤ 256 KiB stored inline in DB |

## 1. Repository layout

```
landrop/
  cmd/landrop/main.go               # serve (default), qr (prints join QR to terminal)
  internal/
    hub/{presence.go (WS rooms), signaling.go, devices.go}
    transfer/{manager.go, relay.go, bitmap.go, verify.go (blake3), zip.go}
    shelf/{shelf.go, sweeper.go, quota.go}
    httpapi/{routes.go, guest.go, ratelimit.go}
    store/{schema.sql, db.go}
    mdns/mdns.go  tlsgen/tlsgen.go  qr/qr.go
  ui/src/
    pages/{Home.tsx (device grid + shelf tab), Transfers.tsx}
    components/{DeviceTile.tsx, SendSheet.tsx (drop/paste/pick), AcceptDialog.tsx,
                ProgressRow.tsx (path indicator direct/via-host), ShelfList.tsx, ShelfItem.tsx,
                TextCard.tsx (copy/open), QRJoin.tsx, RenameSheet.tsx}
    lib/{ws.ts, rtc.ts, relay.ts, chunker.ts, saver.ts (FSA/download strategies), blake3.ts (wasm)}
    sw.ts
  test/{torture/ (transfer scenarios), netns/ (client-isolation compose sim)}
```

## 2. Dependencies

Go: `coder/websocket, modernc.org/sqlite, zeekay/blake3 → decision: lukechampine.com/blake3, grandcat/zeroconf, skip2/go-qrcode, go-chi/chi/v5, klauspost/compress (zip)`. UI: react, zustand, tailwind, `blake3-wasm`, `nanoid`, vite-plugin-pwa. Dev: vitest, playwright (3 browser contexts).

## 3. DB schema

```sql
CREATE TABLE devices (id TEXT PK, name TEXT, emoji TEXT, ua_hint TEXT, room TEXT,
  first_seen INTEGER, last_seen INTEGER, is_guest INTEGER DEFAULT 0);
CREATE TABLE transfers (id TEXT PK, from_device TEXT, to_device TEXT,
  kind TEXT CHECK(kind IN ('files','text','clipboard')), manifest TEXT NOT NULL,
  path TEXT CHECK(path IN ('p2p','relay')), status TEXT CHECK(status IN
  ('offered','accepted','active','done','declined','failed','cancelled')),
  bytes_done INTEGER DEFAULT 0, started_at INTEGER, finished_at INTEGER);
CREATE TABLE shelf_items (id TEXT PK, device_id TEXT, kind TEXT, name TEXT, size INTEGER,
  blob_ref TEXT, text_content TEXT, expires_at INTEGER, downloads INTEGER DEFAULT 0,
  pinned INTEGER DEFAULT 0, created_at INTEGER);
CREATE TABLE settings (key TEXT PK, value TEXT);
```

## 4. Protocol contracts

**WS `/ws` frames:** `hello {deviceToken?} → welcome {device, roomPeers[]}` · `peer_join/peer_leave` · `offer_transfer {to, kind, manifest{files:[{name,size,hash?}], totalSize}} → transfer_offered {transferId}` (recipient gets `incoming_offer`) · `answer_offer {transferId, accept}` · `rtc_signal {transferId, sdp|ice}` (relayed) · `transfer_progress {transferId, bytesDone}` (host-observed on relay; sender-reported P2P) · `rename {name, emoji}`.

**Relay HTTP:** `PUT /api/relay/{tid}/chunks/{n}` (raw bytes, idempotent per n) · `GET /api/relay/{tid}/bitmap` (resume) · `GET /api/relay/{tid}/stream` (ordered chunk stream as they arrive) · `POST /api/transfers/{tid}/complete {blake3}` → server verifies relay-path hash (has all chunks) or trusts P2P receiver's verify report. **Shelf:** `POST /api/shelf` (multipart or `{text}`), `GET /api/shelf` (room+policy filtered), `GET /api/shelf/{id}/download`, `POST /api/shelf/{id}/pin`, `DELETE`. **Join:** `GET /join/{pin?}` QR target → sets cookie for pin rooms; wrong pin → generic 404 page.

**P2P dataflow (`rtc.ts`):** offerer = sender; single reliable ordered DataChannel; frames: 16-byte header `{chunkIndex u64, len u32}` + bytes; receiver acks every 16 chunks (flow control window 32); after last chunk → receiver computes blake3 (wasm, streaming) → `complete`. **Frame endianness: big-endian (network byte order)** for both header fields, pinned regardless of platform — sender writes BE, receiver reads BE (interop test asserts a fixed byte layout). Wi-Fi-drop resume: reconnect WS → `GET bitmap` (relay) or ack-state re-sync (P2P) → continue from first missing chunk (torture-tested).

## 5. Screens/UX contracts

Home: device grid (DeviceTile: emoji, name, platform icon), tap → SendSheet (files multi-pick / drag / paste text / "send clipboard" button using `navigator.clipboard.readText` with permission fallback to paste box); incoming → AcceptDialog (sender, file list, size, Accept/Decline, remember-sender toggle → auto-accept from that device). Transfers page: ProgressRow (speed, ETA, path chip, cancel). Shelf tab: ShelfList (uploader, countdown, pin, download); guests see policy-filtered view. First-visit: name assigned + tiny "how it works" card; QRJoin floating button shows QR of current URL.

## 6. Milestone task lists

**M0** — T1 Go server scaffold + store + WS presence/rooms + auto-names; T2 UI shell + device grid + rename; T3 text/clipboard sends E2E (offer→accept→TextCard); T4 CI + release builds (mac/linux/win/arm64).
**M1** — T1 chunker + relay endpoints + bounded-channel pipe (constant-memory test: 2 GiB transfer < 100 MiB RSS); T2 file offer/accept flow + ProgressRow + cancel; T3 blake3 verify both sides; T4 resume (bitmap) + kill-mid-transfer torture tests; T5 saver.ts strategies + zip batch fallback.
**M2** — T1 rtc.ts + signaling + host-candidates-only; T2 auto-fallback (5 s) + path chip + forced-relay test mode (`?relay=1`); T3 netns/compose client-isolation sim (P2P blocked → relay succeeds); T4 flow control + 100-photo batch test (individual files via FSA, zip fallback elsewhere).
**M3** — T1 shelf (upload, list, expiry sweeper atomic delete + GC test, quota eviction warn→evict); T2 pin; T3 guest policies (pin room cookie, guests-limited filtering tests); T4 QR (terminal + UI).
**M4** — T1 PWA (manifest, share-target POST handler → SendSheet prefilled, install prompt); T2 Web Push opt-in + on-focus offer queue (backgrounded-receiver test); T3 mDNS + `--tls self` + cert QR flow docs; T4 multi-subnet rooms + `--single-room`; T5 docs (NAS/Pi/Docker) + Dockerfile. Tag v1.0.

## 7. Test mapping

Torture suite (chunk loss injection via test hook, disconnects, concurrent senders, 0-byte, 4 GiB, unicode/reserved names) on both paths — release gate. Playwright 3-context E2E incl. forced-relay context. RSS/leak soak under sustained relay. Manual device matrix doc (iOS Safari, Android Chrome) per release.

## 8. Room segmentation, guest enforcement & secure-context UX

**Multi-subnet room segmentation (`hub/presence.go`).** The host may listen on several interfaces (Ethernet + Wi-Fi + Docker bridge). A client is placed in a room keyed by the *server-side network it connected through*, so two clients on the same LAN see each other but a Docker-bridge client doesn't leak into the Wi-Fi room:

```
function roomKeyFor(conn):
  localIP = conn.LocalAddr().IP            # which server interface accepted this connection
  for iface in host.interfaces:
    for net in iface.networks (CIDR):
      if net.contains(localIP): return net.String()   # e.g. "192.168.1.0/24"
  return "default"                          # loopback / unknown → shared default room
# --single-room overrides: every connection → roomKey "single"
```

Loopback (`127.0.0.0/8`) and link-local are folded into `default`. Rooms are ephemeral (no persistence); a device row stores its current `room` for shelf filtering. AC: netns sim proves cross-subnet isolation and same-subnet visibility.

**Guest policy enforcement (`httpapi/guest.go`) — `guests-limited` filtering.** A guest is any device with `is_guest=1` (joined via a PIN room cookie or flagged by policy). Enforcement is a single predicate applied on every shelf/transfer read and write:

```
function canSee(device, item):        # shelf listing filter
  if policy == 'open':          return true
  if policy == 'pin':           return device.room == item.room    # cookie already gated join
  if policy == 'guests-limited':
     if not device.is_guest:    return true                        # full members see all
     return item.device_id == device.id                            # guests: own items only

function canSend(device, toDevice):
  if policy != 'guests-limited': return true
  return device.is_guest ? true : true      # sending TO guests always allowed
  # guests may send (send-only); receiving is restricted by canSee on the shelf/offer
```

Offers to a guest that would require them to *read* a non-owned item are rejected server-side (not just hidden in UI) — tested in the guests-limited suite.

**Secure-context fallback UX (WebRTC).** P2P is attempted only when `isSecureContext` is true (HTTPS or `--tls self`). On plain HTTP the UI never shows a broken "direct" attempt: the path chip renders **"via host"** from the start and a subtle info tooltip explains "Direct transfer needs HTTPS — run `landrop --tls self` for peer-to-peer." No error, no retry loop; relay is the silent, guaranteed floor. When secure context is available but P2P setup times out (5 s), the chip flips from "direct…" to "via host" once, with a one-line toast "couldn't connect directly, using host relay."
