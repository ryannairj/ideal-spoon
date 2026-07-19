# Implementation Spec — Photo Memory

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Product name | **Snapmind** |
| License | proprietary; self-host compose still shipped |
| Stack | Next.js 15 PWA + TS 5; Postgres 16 + pgvector (Drizzle); Redis + BullMQ worker; S3-compatible store (MinIO in compose) |
| Vision/LLM | processing: `claude-haiku-4-5` vision; Ask answering: `claude-sonnet-5`; embeddings `text-embedding-3-small` (1536 d); all behind `packages/llm`; self-host degraded mode: `tesseract.js` OCR + no captions |
| Offline outbox | IndexedDB via `idb`, queue table `outbox`, background sync via service worker `sync` event + fallback interval |
| Image handling | client resizes to max 2048 px JPEG q85 before upload (originals NOT kept — decision: resized is the original); thumbs 400 px generated server-side (`sharp`) |
| Auth | Auth.js v5 magic link; secrets reveal re-auth via WebAuthn (`@simplewebauthn`) optional, fallback = re-enter email code |
| Entities schema | fixed key set: `ssid, password, code, plate, level, zone, color_code, brand, model, serial, date, amount, phone, url, other[]`; `secret:true` flag on password/code |
| Kinds | `parking, document, label, receipt, whiteboard, ticket, other` |
| Push | web-push VAPID |
| Ports | web 3000, worker 3003 |

## 1. Repository layout

```
snapmind/
  src/
    app/
      page.tsx                    # home: active pins rail + capture FAB + recent grid
      capture/page.tsx            # camera-first (opens getUserMedia immediately)
      m/[id]/page.tsx             # memory detail
      ask/page.tsx                # ask UI + recent asks
      library/page.tsx            # filters + virtualized grid
      settings/page.tsx           # privacy, cloud-enhance toggle, export/delete
      api/{upload/route.ts, memories/[id]/route.ts, ask/route.ts,
           reminders/route.ts, export/route.ts}
    components/{CaptureView, OutboxBadge, MemoryCard, EntityField (copy btn, masked when secret),
                PinCard, AskBox, AnswerCard, ReminderSheet, TagChips, GridVirtual}
    lib/{outbox.ts, sw-register.ts, api.ts}
    sw.ts
  worker/src/{index.ts, process.ts (pipeline), remind.ts, gc.ts}
  packages/db/src/schema.ts  packages/llm/src/{provider.ts, prompts/{process.ts, ask.ts}}
  fixtures/images/ (30 labeled)  fixtures/eval/{memories.json, questions.yaml}
  docker-compose.yml (pg, redis, minio, web, worker)
```

## 2. Dependencies

`next, react, drizzle-orm, postgres, bullmq, ioredis, @aws-sdk/client-s3, sharp, idb, web-push, zod, tailwindcss, @tanstack/react-virtual, exifr` (client EXIF), `@simplewebauthn/browser|server`, `tesseract.js` (degraded mode). Dev: vitest, playwright.

## 3. Configuration

`DATABASE_URL, REDIS_URL, S3_ENDPOINT/KEY/SECRET/BUCKET, ANTHROPIC_API_KEY, OPENAI_API_KEY (embeddings), VAPID_*, CLOUD_ENHANCE_DEFAULT=true, SELF_HOST_MODE=false`. Per-user setting `cloud_enhance` overrides default.

## 4. Database schema

```sql
CREATE TABLE users (id uuid PK, email text UNIQUE, cloud_enhance boolean DEFAULT true,
  webauthn jsonb, created_at timestamptz);
CREATE TABLE memories (id uuid PK, user_id uuid NOT NULL,
  image_key text NOT NULL, thumb_key text, taken_at timestamptz NOT NULL,
  title text, caption text, ocr_text text, entities jsonb DEFAULT '{}',
  tags text[] DEFAULT '{}', kind text DEFAULT 'other',
  pinned_until timestamptz, note text,
  status text DEFAULT 'processing' CHECK (status IN ('processing','ready','failed')),
  embedding vector(1536), tsv tsvector GENERATED ALWAYS AS
    (to_tsvector('english', coalesce(title,'')||' '||coalesce(caption,'')||' '||coalesce(ocr_text,'')||' '||coalesce(note,''))) STORED,
  created_at timestamptz DEFAULT now());
CREATE INDEX memories_vec ON memories USING hnsw (embedding vector_cosine_ops);
CREATE INDEX memories_tsv ON memories USING gin (tsv);
CREATE TABLE reminders (id uuid PK, memory_id uuid REFERENCES memories ON DELETE CASCADE,
  due_at timestamptz NOT NULL, note text, fired_at timestamptz, snoozed_from uuid);
CREATE TABLE asks (id uuid PK, user_id uuid, query text, answer text,
  memory_ids uuid[], confidence real, at timestamptz DEFAULT now());
CREATE TABLE push_subs (id uuid PK, user_id uuid, endpoint text UNIQUE, keys jsonb);
```

## 5. API contract

| Route | Behavior |
|---|---|
| `POST /api/upload` multipart `{image, takenAt, note?, tags?}` | creates memory `processing`, S3 put, enqueue job → `{memoryId}` (idempotency key header from outbox id) |
| `GET /api/memories?cursor&kind&tag&q` | keyset-paginated grid data |
| `GET/PATCH/DELETE /api/memories/{id}` | detail / edit title,tags,entities,note,kind / hard delete (S3 objects too) |
| `POST /api/memories/{id}/pin` `{until?}` / `DELETE …/pin` | active-situation pin |
| `POST /api/ask` `{query}` | SSE: retrieval → `{answer, citations:[memoryId], confidence}`; logs to asks |
| `POST /api/reminders` `{memoryId, dueAt, note}` / `PATCH` (snooze) / `DELETE` | |
| `GET /api/export` | zip stream (images + JSON) |
| `POST /api/push/subscribe` | |

## 6. Processing pipeline (`worker/process.ts`)

```
job(memoryId): fetch image from S3 → sharp thumb →
 if cloud_enhance: vision prompt (single call) → strict JSON
   {title ≤8w, caption, ocr_text, entities{fixed keys}, kind, tags[≤5]}
   zod-validate; 1 retry with error feedback; fail → status ready-with-degradation:
   (title = "Photo " + date, rest null) + mark degraded flag in entities._degraded
 else: tesseract OCR → ocr_text only; title from date
 embed(title+caption+ocr_text) → pgvector
 kind-based automations: parking/ticket → suggest pin (push "Pin this?" action);
   entities.date future → create suggested reminder row (unconfirmed flag)
 status='ready'; notify client via push-or-poll
```
P50 < 15 s budget: single vision call, thumb parallel with LLM.

## 7. Ask flow

Hybrid retrieve: top 8 vector + top 8 FTS (websearch_to_tsquery) → RRF → top 4 → answer model sees {query, per-memory: title/caption/ocr/entities/thumb URL as image attachment for top 2} → JSON `{answer, memoryIds, confidence}`. Confidence < 0.4 → "couldn't find it" template (never guess). Secrets: if answer would reveal a `secret:true` entity, respond with masked value + "tap to reveal" (client re-auth then reads entity directly).

## 8. Screens

Home: pinned rail (PinCard: big, one-tap open, Done button), AskBox, recent 12 grid, capture FAB. Capture: full-screen camera (rear default), shutter, burst (≤5), note field + recent-tag chips post-shot, offline badge. Detail: image (pinch zoom), EntityField list w/ copy buttons + masked secrets, tags editor, reminder sheet, pin toggle, delete. Library: kind filter tabs, tag chips, date range, search box (FTS), virtualized grid. Ask: query → AnswerCard (answer + cited memory thumbnails). Settings: cloud-enhance explainer + toggle, export, delete account, privacy text ("exactly what leaves the server" list).

## 9. Milestone task lists

**M0** — T1 scaffold + auth + schema + compose; T2 capture page (getUserMedia, resize, EXIF takenAt via exifr) + upload route + S3; T3 outbox (idb queue, sw sync, OutboxBadge) + airplane-mode test; T4 library grid basic + detail; T5 CI.
**M1** — T1 worker + pipeline + zod schema + retry/degrade paths; T2 EntityField UI + edit PATCH; T3 fixture regression suite (30 images, recorded vision fixtures; router-sticker case asserts ssid+password extracted); T4 thumbs + processing status polling.
**M2** — T1 embeddings + hybrid retrieval + RRF; T2 ask route (SSE) + AnswerCard + asks log; T3 eval harness (`fixtures/eval`: seed 30, 20 questions incl. 5 unanswerable; CI recorded ≥85% top-3 / 0 fabrications, nightly live); T4 secret masking + WebAuthn reveal.
**M3** — T1 pins (suggest push, PinCard, auto-expiry job); T2 reminders CRUD + remind.ts (due scan 1 min, push w/ thumbnail, snooze) + fake-clock tests; T3 suggested reminders from date entities.
**M4** — T1 virtualized 2k-item grid perf pass; T2 selective import (file picker ≤50, same pipeline); T3 export zip + hard-delete + GC job (orphan S3 scan) + tests; T4 SELF_HOST_MODE compose profile (tesseract path) + privacy settings page; T5 iOS Safari capture/push verification checklist executed. Tag v1.0.

## 10. Test mapping

AC-named vitest/playwright specs per PLAN F1–F7. Release-blocking: outbox offline E2E, hard-delete-no-orphans, eval fabrication=0. Cost metering test asserts ≤ 2 model calls per photo.
