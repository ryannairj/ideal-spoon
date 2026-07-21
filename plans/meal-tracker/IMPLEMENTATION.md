# Implementation Spec — Plately (meal tracker)

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Plately** · proprietary; self-host compose shipped |
| Stack | Next.js 15 PWA + TS 5; Postgres 16 + pgvector (Drizzle); Redis + BullMQ; S3-compatible (MinIO in compose); Auth.js magic link |
| Vision model | `claude-haiku-4-5` for estimation; embeddings: image via `nomic-embed-vision` API-compat NOT used — decision: text-side embedding of `{caption from estimation + note}` with `text-embedding-3-small` + perceptual hash (`sharp` dHash) for repertoire pre-filter; cosine `@TUNE(τ=0.86)` AND dHash distance ≤ `@TUNE(dHashMax=10)` → candidate match. Both calibrated in M2 against the repertoire precision/recall fixture sets (same-meal-different-lighting must match; lookalike-different-meal must not) |
| Estimation output (zod, fixed) | `{items:[{name, portion_desc, grams_est, kcal_lo, kcal_hi, protein_g, carbs_g, fat_g, confidence}], meal_confidence, caption}` |
| Display rule | always show `kcal_lo–kcal_hi`; rollups use midpoint; per-user calibration multiplier applied to both bounds |
| Slots | inferred: <10:30 breakfast, <15:00 lunch, <21:00 dinner, else snack (user tz; editable) |
| Targets | Mifflin-St Jeor × activity factor {1.2,1.375,1.55,1.725} + mode adjust {maintain 0, cut −20%, bulk +10%}; protein target 1.6 g/kg default |
| Streak | ≥2 meals logged/day; 2 grace tokens/month auto-applied; hide-numbers mode flag |
| Barcode | `@zxing/browser`; OpenFoodFacts API v2, cache table |
| Corrections | chips: remove item, ×0.5, ×1.5, ×2, "no <ingredient>" free text; local recompute for portion ops, LLM only for free-text swaps |
| Ports | web 3000, worker 3005 |

## 1. Repository layout

```
plately/
  src/
    app/
      (app)/{today/page.tsx, log/page.tsx (camera), meal/[id]/page.tsx,
             history/page.tsx, reports/page.tsx, settings/page.tsx, onboarding/page.tsx}
      api/{meals/route.ts, meals/[id]/route.ts, estimate/…, barcode/route.ts,
           repertoire/…, weights/route.ts, export/route.ts, push/…}
    services/{estimator.ts, repertoire.ts, calibration.ts, targets.ts, streaks.ts, reportgen.ts}
    components/{CameraCapture, DayRing, ProteinBar, MealCard (range display), ItemChipEditor,
                UsualBanner ("the usual?"), ConfidenceQ (S/M/L bowl question), BarcodeScan,
                PortionSlider, StreakBadge, WeeklyReport, HideNumbersToggle}
    lib/{outbox.ts (idb), sw.ts}
  worker/src/{index.ts, estimate.ts, weekly.ts, retention.ts}
  packages/db/schema.ts
  fixtures/meals/ (40 labeled photos + ground-truth.yaml)  fixtures/llm/
  docker-compose.yml
```

## 2. Dependencies

`next, react, drizzle-orm, postgres, bullmq, ioredis, @aws-sdk/client-s3, sharp, zod, idb, web-push, @zxing/browser, date-fns-tz, tailwindcss, recharts (reports), exifr`. Dev: vitest, playwright.

## 3. Database schema

```sql
CREATE TABLE users (id uuid PK, email text UNIQUE, tz text, units text DEFAULT 'metric',
  targets jsonb, calibration real DEFAULT 1.0, hide_numbers boolean DEFAULT false,
  streak jsonb DEFAULT '{}', created_at timestamptz);
CREATE TABLE meals (id uuid PK, user_id uuid NOT NULL, at timestamptz NOT NULL, slot text,
  photo_key text, note text, items jsonb NOT NULL DEFAULT '[]',
  kcal_lo int, kcal_hi int, protein_g real, carbs_g real, fat_g real,
  source text CHECK (source IN ('vision','repertoire','barcode','manual')),
  confidence real, corrected boolean DEFAULT false, repertoire_id uuid,
  status text DEFAULT 'ready' CHECK (status IN ('estimating','ready','failed')),
  created_at timestamptz DEFAULT now());
CREATE INDEX meals_user_day ON meals(user_id, at);
CREATE TABLE repertoire (id uuid PK, user_id uuid, label text, embedding vector(1536),
  dhash bigint, base_items jsonb, times_logged int DEFAULT 0, last_at timestamptz,
  adjustments jsonb DEFAULT '{}', denied_pairs jsonb DEFAULT '[]');
CREATE INDEX repertoire_vec ON repertoire USING hnsw (embedding vector_cosine_ops);
CREATE TABLE products (barcode text PK, name text, per100g jsonb, source text, cached_at timestamptz);
CREATE TABLE weights (user_id uuid, date date, kg real, PRIMARY KEY(user_id, date));
CREATE TABLE push_subs (…);
```

## 4. API contract

| Route | Behavior |
|---|---|
| `POST /api/meals` multipart `{photo?, note?, at?}` | resize client-side 1600 px; creates meal `estimating`, enqueue → `{mealId}`; idempotency header from outbox |
| `GET /api/meals?date=` / `GET /api/meals/{id}` | day view / detail |
| `PATCH /api/meals/{id}` `{items?|slot?|note?}` | corrections; portion ops recompute server-side deterministically; sets corrected, updates repertoire adjustments |
| `POST /api/meals/{id}/correct-text` `{text}` | LLM re-estimate of named items only (delta prompt) |
| `POST /api/meals/{id}/usual-response` `{accept: bool}` | accept → apply profile; deny → push (mealId, repertoireId) to denied_pairs + τ penalty for pair |
| `POST /api/barcode` `{barcode, grams}` | OFF lookup (cache-first) → meal source=barcode |
| `GET /api/reports/week?start=` | computed report JSON |
| `POST /api/weights {date, kg}` · `GET /api/export.csv` | |

## 5. Estimation & repertoire flow (worker `estimate.ts`)

```
job(mealId): photo → dHash + prelim: embed(note) if note else skip-to-vision
  candidates = repertoire where user, cosine ≥ .86 (on caption-embedding) AND dhash ≤ 10
    AND (mealId,repId) ∉ denied_pairs AND times_logged ≥ 2
  if best candidate → apply base_items ⊕ adjustments → source='repertoire', confidence=candidate score
    → UsualBanner state (client shows "the usual? ✓/✗")
  else → vision call (photo + note, prompt `fixtures/prompts/estimate.md`) → zod validate (retry 1 with error) → items + caption
    → meal_confidence < `@TUNE(0.45)` → flag needs_question (client shows ConfidenceQ S/M/L → scale grams ×{0.7,1,1.4} locally)
    → both attempts fail zod/timeout → status='failed': meal saved as empty manual-entry shell so nothing is lost; client opens ItemChipEditor with a "couldn't read this photo — add items" banner
  calibration: kcal_lo/hi ×= users.calibration
  embed caption → on user confirm (2nd identical log), upsert repertoire row
weekly.ts: correction bias = Σ(user-corrected kcal midpoint / estimated) capped `@TUNE([0.85,1.15])`
  — the cap bounds a single week's calibration drift to ±15% so one mislabeled meal can't swing targets wildly
  → users.calibration (EMA `@TUNE(α=0.3)`); surfaced in report ("we've adjusted by −8%")
retention: photos deleted after N days if user sets photo_retention (default keep)
```

## 6. Screens

Today (DayRing kcal range fill, ProteinBar, meal list by slot, streak badge, log FAB) · Log (camera-first, note field, offline queue badge, barcode button) · Meal detail (photo, ItemChipEditor rows: name, grams, kcal range; correction chips; UsualBanner; ConfidenceQ when flagged) · History (calendar + day drill-in) · Reports (weekly: intake trend vs weights line, protein consistency bars, top meals, calibration note) · Onboarding (3 steps: stats → activity/mode → targets shown editable) · Settings (targets, hide-numbers, retention, export, delete).

## 7. Milestone task lists

**M0** — T1 scaffold + auth + schema + compose; T2 camera capture + resize + upload + outbox offline; T3 manual meal entry + Today view + targets.ts (unit tests vs published Mifflin examples) + onboarding.
**M1** — T1 worker estimation + zod + retry/degrade; T2 MealCard ranges + detail + ItemChipEditor local recompute (<500 ms test); T3 ConfidenceQ flow; T4 fixture benchmark harness (40 photos: recorded fixtures in CI; metric script: midpoint within ±25% for ≥70% — publish `BENCHMARK.md`).
**M2** — T1 repertoire matching (dHash + embedding + thresholds) + UsualBanner accept/deny + denied_pairs; T2 3-identical-photos fixture test (3rd auto-matches; deny blocks 4th); T3 calibration.ts + EMA + report surfacing + transparency copy.
**M3** — T1 streaks (fake-clock, tz/DST, grace tokens) + hide-numbers mode (ring fills only); T2 weekly report page + email opt-in; T3 barcode scan + OFF client + cache + PortionSlider; T4 offline hardening E2E.
**M4** — T1 in-app benchmark/accuracy page (from BENCHMARK.md data); T2 CSV export (every field); T3 retention job + hard delete; T4 cost metering (< $0.01/day/user at 3 meals — assert ≤1 vision call per non-repertoire meal); T5 Stripe stub behind flag; polish. Tag v1.0.

## 8. Test mapping

Benchmark harness = F2 gate (CI recorded, nightly live w/ cost). Repertoire precision/recall on fixture sets (same meal diff lighting vs lookalike different meals). Fake-clock suites (slots/streaks/reports). ED-adjacency review: copy audit checklist committed (`docs/tone-checklist.md`), hide-numbers E2E.

## 9. Prompt fixtures & Error Recovery

**Prompt fixtures** (`fixtures/prompts/`, mirrored under `fixtures/llm/` for recorded responses, versioned `# v1`):
- `estimate.md` — full estimation; vars `{{imageAttachment}} {{note}}`; output = the Estimation zod schema (§0); worked example included.
- `correct-text.md` — free-text correction delta; vars `{{currentItems}} {{userText}}`; output = revised `items[]` only (never re-reads the photo).

**Error Recovery & Graceful Degradation**

| Failure | Trigger | Backoff | Fallback | User-facing UX |
|---|---|---|---|---|
| Vision estimation | non-JSON / zod-invalid / timeout 30s | 1 retry with validation error appended | `status='failed'`, save empty manual shell; nothing lost | Meal detail banner "couldn't read photo — add items" |
| Vision provider down | 5xx/network, both attempts | BullMQ 3× base 10s ×3 then dead-letter | same manual-shell fallback | as above |
| Free-text correction | LLM fail/invalid | 1 retry | keep prior items unchanged | toast "couldn't apply, try again" |
| Embedding | 5xx/timeout | 2 retries base 2s ×2 | meal saved without repertoire embedding (no "usual?" matching); backfilled nightly | silent |
| Barcode (OpenFoodFacts) | 404 / 5xx / timeout 8s | no retry on 404; 1 retry on 5xx | fall back to manual grams+macros entry | "product not found — enter manually" |
| Upload | S3/network fail | outbox exp backoff base 5s cap 5min | stays queued, nothing lost | offline badge shows pending count |
