# Photo Memory — remember anything by photographing it

**Category:** Life improvement · **Difficulty:** Medium · **Platform:** Mobile-first PWA (native wrapper later)

## 1. Vision

Where did I park? Which fuse is the kitchen one? What wine did we love? What's my bike lock combo hint? You already solve this by taking photos — then they drown among 30k camera-roll shots. Photo Memory is a *separate, tiny* capture app: snap → it OCRs, describes, and files the photo → later you just ask ("what paint color is the hallway?") and get the photo + answer back.

## 2. Why it can win

- Camera roll search (Google/Apple) is passive and mixed with your whole life; this is an **intentional memory vault** — everything in it was captured *to be remembered*, so recall is high-precision.
- Ask-in-natural-language + answer-extracted-from-photo ("Hallway is 'Dulux Natural White'") beats scrolling thumbnails.
- Reminders attach to memories: "boarding pass — remind me at 6am"; "parking spot — remind in 2h".

## 3. Users & use cases

Everyone; especially forgetful busy people, renters (meter readings, condition photos), travelers.

1. Snap parking level sign → auto-titled "Parking – Level 3, Zone B" → pinned as *active situation* → one tap to recall, auto-expires when you return.
2. Snap wine label at dinner → later ask "that red wine from Sofia's birthday".
3. Snap router sticker → ask "wifi password" → answer text surfaced, copyable.
4. Snap medicine box + set reminder "reorder in 3 weeks".

## 4. MVP scope & non-goals

**MVP:** capture (camera/upload), auto-processing (OCR + vision caption + tags + entity extraction), library with filters, natural-language ask, active-situation pins, reminders, PWA installable, accounts, E2E-ish privacy posture (see F7).
**Non-goals:** importing your whole camera roll (defeats the vault premise; allow selective import ≤ 50 at once), social/sharing (single-user), video, native apps in MVP (PWA with camera works; wrap with Capacitor in v1.1 if share-sheet/offline demands it).

## 5. Tech stack

- **App:** Next.js 15 PWA + TypeScript; IndexedDB outbox for offline capture (photos queue and upload when online).
- **API/store:** Postgres + pgvector (Drizzle); S3-compatible object storage for images (originals + thumbs).
- **Processing worker:** Node + BullMQ: vision LLM (captioning, entity extraction, Q&A) via provider-agnostic module; OCR primary via vision model, `tesseract` fallback for offline/self-host mode.
- **Search:** hybrid — pgvector embeddings (caption+OCR text) + Postgres FTS; recency/pin boosting.

## 6. Architecture

```
PWA (capture, offline outbox) → API → S3 (image) + jobs queue
                                        └ worker: OCR + caption + entities + embedding → Postgres
ask: query → hybrid retrieve top-k memories → vision/LLM answer over winners → answer + citations(photos)
```

## 7. Data model

- `users(id, email, …)` 
- `memories(id, user_id, image_key, thumb_key, taken_at, title, caption, ocr_text, entities_json /*{brand, color_code, ssid, plate, dates…}*/, tags text[], kind{parking,document,label,receipt,whiteboard,other}, pinned_until, expires_at, embedding vector, status{processing,ready,failed})`
- `reminders(id, memory_id, due_at, note, fired_at)`
- `asks(id, user_id, query, answer, memory_ids, at)` — recall log, also powers "recently asked".

## 8. Feature specs

**F1 — Capture.** Open-to-shutter < 2 s (PWA launches straight into camera); burst up to 5; optional one-line note + tag chips (recent tags suggested); works offline (outbox with visible queue). *AC:* airplane-mode capture syncs correctly later; EXIF timestamp preserved as `taken_at`.

**F2 — Auto-processing.** Worker generates: title (≤ 8 words), caption, OCR text, entity extraction into typed fields, kind classification, tags. *AC:* processing P50 < 15 s; failure leaves memory usable (image + manual title) and retryable; router-sticker fixture yields extractable SSID + password entities.

**F3 — Ask.** Free-text question → top-k retrieve → answer with the source photo(s) inline; answer text copyable; "not found" honesty below confidence threshold. *AC:* eval set (30 seeded memories, 20 questions) ≥ 85% correct-photo retrieval in top-3; zero fabricated answers on the 5 unanswerable eval questions.

**F4 — Library.** Reverse-chron grid; filter by kind/tag/date; full-text search fallback; detail view shows extracted fields with copy buttons; edit title/tags/fields. *AC:* 2k memories scroll smoothly (virtualized) on a mid-range phone.

**F5 — Active situations.** Kinds like parking/luggage-locker/hotel-room auto-suggest pinning; pinned card sits on top of home until tapped "done" or auto-expiry. *AC:* parking photo → pin appears without user action beyond accepting the suggestion.

**F6 — Reminders.** Attach to any memory: at time / in duration / smart suggestions from entities (expiry dates on tickets/food). Web-push + email fallback. *AC:* reminder fires with photo thumbnail in the notification; snooze works.

**F7 — Privacy posture.** Images encrypted at rest server-side; processing uses external LLM only if user enables "cloud enhance" (default on, clearly explained; self-host mode = tesseract + local caption model optional, degraded). Delete = hard delete incl. blobs. *AC:* delete leaves no orphaned S3 objects (GC test); privacy settings page states exactly what leaves the server.

## 9. Milestones

- **M0 (1):** scaffold, auth, capture→upload→library (no AI yet), offline outbox.
- **M1 (2):** processing worker (OCR/caption/entities), detail view with copyable fields. *Ships: useful vault.*
- **M2 (3):** hybrid search + Ask with citations + eval harness. *Ships: MVP wow-demo.*
- **M3 (4):** active situations, reminders, push. 
- **M4 (5):** polish (virtualized grid, tag UX, selective import), self-host docker-compose, privacy page. *Ships: v1.0.*

## 10. Testing

- Fixture image set (30 images: labels, signs, whiteboards, receipts, low-light) with labeled expected entities → processing regression suite (recorded LLM fixtures in CI, live nightly).
- Ask eval harness as F3 metric gate.
- Offline outbox: Playwright with network emulation (capture offline → sync).
- Object-storage GC and hard-delete tests.

## 11. Risks & open questions

- iOS PWA camera/push limitations → verify capture UX on iOS Safari in M0; Capacitor wrapper is the planned escape hatch, not a rewrite.
- Vision API cost per photo → batch small, cache, and keep "cloud enhance" per-user metering; target < $0.01/photo.
- Sensitive content (passwords, documents) → encourage app-lock (WebAuthn re-auth for revealing extracted secrets; `entities.secret=true` fields masked by default).
