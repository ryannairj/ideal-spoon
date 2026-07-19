# Plately — photo-first meal tracking without the database tedium

**Category:** Life improvement · **Difficulty:** Medium · **Platform:** Mobile-first PWA

## 1. Vision

Calorie apps die in the logging friction: search a database, weigh ingredients, pick 1 of 40 "chicken breast" entries. Plately's bet: **a photo (plus an optional one-line note) is enough**. Vision LLM estimates the meal, portions, and macros with an honest confidence range; you correct with taps, and it *learns your repertoire* — after two weeks, "the usual overnight oats" logs itself from one snap at 95% confidence. Aimed at awareness and consistency, not gram-perfect bodybuilding prep.

## 2. Why it can win

- Photo estimation exists (and is fuzzy) but nobody pairs it with (a) honest uncertainty ranges instead of fake precision, and (b) a **personal food memory** that converges to *your* actual recurring meals — which is where accuracy really comes from.
- Positioning against MFP-style apps: zero database spelunking, no ads, streak-driven consistency (the metric that predicts outcomes anyway).

## 3. Users & use cases

People wanting awareness/weight management without prep-cook rigor.

1. Snap lunch → "Chicken burrito bowl, ~650–800 kcal, P 40 g" → tap ✓ (3 seconds total).
2. Correction: "no cheese, double rice" typed or tapped → estimate updates, correction remembered for this repertoire item.
3. Evening: day ring vs target, macro split, streak.
4. Weekly report: trends, protein consistency, "your estimates vs corrections" calibration note.
5. Barcode fallback for packaged foods (OpenFoodFacts).

## 4. MVP scope & non-goals

**MVP:** photo capture + text-note logging, LLM estimation with ranges, quick-correct UX, personal repertoire learning, daily targets & rings, streaks, weekly report, barcode scan (OpenFoodFacts), history/search, export CSV, offline capture queue.
**Non-goals:** micronutrients (macro + kcal only), recipes/meal planning (see unrecipe plan), social, wearab/le weight-scale integrations (v1.1), medical claims of accuracy (explicit in-app framing), native apps.

## 5. Tech stack

- **App:** Next.js 15 PWA + TypeScript; IndexedDB offline outbox for photos.
- **Backend:** Postgres + pgvector (Drizzle), S3-compatible image store, BullMQ worker.
- **Estimation:** vision LLM (provider-agnostic) with structured output `{items[{name, portion_desc, grams_est, kcal_range, protein_g, carbs_g, fat_g, confidence}], meal_confidence}`; repertoire matching via image+text embeddings before full estimation (cache hit = cheap + instant).
- **Barcode:** `@zxing/browser` client-side + OpenFoodFacts API with local cache table.

## 6. Architecture & estimation flow

```
snap → embed(image + note) → repertoire match?
  hit (cos ≥ τ, user-confirmed ≥ 2 times) → apply stored profile (instant, editable)
  miss → vision LLM estimate (ranges) → user confirm/correct
corrections → update repertoire item (name, per-item deltas, learned portion) 
daily rollup uses range midpoints; UI always shows the range
```

## 7. Data model

- `users(id, email, targets_json /*kcal, protein, mode{maintain,cut,bulk}*/, units, streak_state)`
- `meals(id, user_id, at, slot{breakfast,lunch,dinner,snack} /*inferred from time, editable*/, photo_key NULLABLE, note, items_json, kcal_lo, kcal_hi, protein_g, carbs_g, fat_g, source{vision,repertoire,barcode,manual}, confidence, corrected bool)`
- `repertoire(id, user_id, label, embedding, base_items_json, times_logged, last_at, learned_adjustments_json)`
- `products(barcode, name, per100g_json, source, cached_at)`
- `weights(user_id, date, kg)` — optional manual weigh-ins for trend vs intake chart.

## 8. Feature specs

**F1 — Capture & log.** Camera-first screen; optional note field; works offline (queue + placeholder entry). *AC:* snap→logged (repertoire hit) < 3 s perceived; offline meals sync with correct timestamps.

**F2 — Estimation with honesty.** Structured ranges as §5; UI shows "650–800 kcal", never a fake "728". Low-confidence (< threshold) prompts one clarifying question max ("roughly how big was the bowl? S/M/L"). *AC:* schema-validated outputs (reject/retry malformed); fixture set of 40 labeled meal photos: midpoint within ±25% of ground truth for ≥ 70% (this is the product's honesty bar — measured, published in-app).

**F3 — Quick corrections.** Tap an item → remove/halve/double chips + free text; corrections re-estimate deltas locally when possible (item swap → LLM only if needed). *AC:* common corrections (remove item, portion ±) apply < 500 ms without a model call.

**F4 — Repertoire learning.** After 2 confirmed logs of a matched meal, auto-apply profile with "the usual?" banner (one tap to accept/deny — denial lowers τ for that item). Repertoire manager screen (rename, merge, delete). *AC:* fixture sequence of 3 identical-meal photos: third is auto-matched; false-match denial prevents 4th auto-apply of that pairing.

**F5 — Targets, rings, streaks.** Onboarding computes targets (Mifflin-St Jeor + activity + mode, editable); day ring (kcal) + protein bar; streak = logged ≥ 2 meals/day (consistency metric, not perfection); grace tokens (2/month) to keep streaks humane. *AC:* target math unit-tested against published examples; streak logic fake-clock tested incl. timezones/DST.

**F6 — Reports & data.** Weekly email/report page: intake trend vs weight trend (if logged), protein consistency, top meals, calibration note ("you corrected estimates down 8% on average — we've adjusted"). Global correction bias feeds a per-user calibration multiplier. *AC:* calibration multiplier applied to future estimates and shown transparently; CSV export includes every field.

**F7 — Barcode fallback.** Scan → per-100 g data → portion slider. *AC:* known-barcode fixture scans resolve < 2 s; unknown barcodes offer manual entry with the photo attached.

## 9. Milestones

- **M0 (1):** scaffold, auth, capture + manual logging, day view + targets.
- **M1 (2):** vision estimation pipeline + ranges + corrections. *Ships: core magic.*
- **M2 (3):** repertoire matching/learning + calibration. *Ships: the moat.*
- **M3 (4):** streaks, weekly reports, barcode, offline hardening.
- **M4 (5):** polish, accuracy benchmark page, export, billing stub if SaaS. *Ships: v1.0.*

## 10. Testing

- Labeled photo benchmark (40 meals, weighed ground truth — build once, reuse forever) gating F2; recorded-LLM fixtures in CI, nightly live accuracy run with cost log.
- Repertoire matcher: precision/recall on fixture photo sets (same meal different lighting/angles vs similar-looking different meals).
- Fake-clock suites (slots, streaks, reports); offline outbox E2E.

## 11. Risks & open questions

- Accuracy ceiling of photo estimation → the product narrative is ranges + trends + consistency; the in-app benchmark page keeps us honest and defuses "it said 728 and was wrong".
- Eating-disorder adjacency → no shaming copy, streaks reward logging not restriction, configurable hide-numbers mode (show only ring fill) — include from MVP.
- Vision cost → repertoire cache + cheap models; target < $0.01/day/user at 3 meals.
