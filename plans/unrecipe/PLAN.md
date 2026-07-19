# Unrecipe — recipes without the life story, plus a pantry that plans dinner

**Category:** Life improvement · **Difficulty:** Medium · **Platform:** Mobile-first PWA

## 1. Vision

Paste any recipe URL and get just the recipe — clean ingredients, clean steps, no 2,000-word childhood memoir, no popups — saved forever in *your* box. Then the compounding layer: your saved recipes know your pantry staples, scale servings properly, merge into one de-duplicated shopping list, and answer the eternal question "what can I make tonight with what's here?"

## 2. Why it can win

- Recipe savers exist (Paprika et al.) but are aging, paid-per-platform, and their extraction breaks on modern JS-heavy sites; LLM-era extraction is a step change (works on *anything*, including screenshots of grandma's card and TikTok caption text).
- The pantry-match feature ("cookable now: 7 recipes, missing-1-ingredient: 5") turns a filing cabinet into a daily-use tool.

## 3. Users & use cases

Home cooks; meal-preppers; anyone with 90 screenshots of recipes they'll never find again.

1. Share a URL from the browser → clean recipe card in your box in seconds.
2. Snap a cookbook page / paste a TikTok caption → same clean card.
3. Sunday: pick 4 dinners → one shopping list, grouped by aisle, staples you already have auto-checked off.
4. Weeknight: "cookable now" shelf based on pantry; tap → cook mode (huge text, step-by-step, screen stays on, timers inline).
5. Scale 4 → 7 servings; metric⇄imperial toggle.

## 4. MVP scope & non-goals

**MVP:** capture (URL, photo/OCR, raw text paste, share-target); LLM normalization to a strict recipe schema; recipe box (tags, search, favorites); cook mode with wake-lock + inline timers; scaling + unit conversion; pantry (staples checklist + rolling "have it" items); shopping list (multi-recipe merge, aisle grouping, check-off, share as text); cookable-now matching.
**Non-goals:** social/publishing (private box only; share = export a card image/text), nutrition facts (out of MVP; meal-tracker plan owns that), meal-plan calendars with drag-drop weeks (simple "this week" picks only), grocery-store API ordering, comments/ratings beyond a private note + "made it" counter.

## 5. Tech stack

- **App:** Next.js 15 PWA + TypeScript; offline-capable box (recipes cached in IndexedDB; capture queues offline).
- **Backend:** Postgres (Drizzle) + S3 (source snapshots, photos); BullMQ extraction workers.
- **Extraction ladder (cost-aware):** 1) JSON-LD/microdata (`schema.org/Recipe`) — free and exact when present → 2) readability-extracted DOM → cheap LLM normalize → 3) vision LLM for photos/screenshots. All paths emit the same strict schema, validated (zod), with per-field confidence; ingredient lines additionally parsed to `{qty, unit, item, prep, optional}` for scaling/matching.
- **Matching:** canonical ingredient dictionary (~800 items + aliases, versioned YAML) → embeddings fallback for unseen items.

## 6. Architecture

```
capture → snapshot source (html/screenshot stored — provenance & re-extraction)
→ extraction ladder → normalized recipe (schema-validated) → user review-lite (only low-confidence fields flagged) → box
pantry + box → matcher (canonical ids) → cookable-now shelf / missing-ingredient deltas → shopping list builder
```

## 7. Data model

- `users(id, email, units_pref, region)` 
- `recipes(id, user_id, title, source{url,photo,text}, source_ref, image_ref, servings, total_min, ingredients_json /*parsed lines*/, steps_json /*[{text, timer_sec?}]*/, tags[], note_md, made_count, favorite, confidence_json, created_at)`
- `ingredient_dict(id, canonical, aliases[], aisle, is_common_staple)` 
- `pantry(user_id, ingredient_id, state{always_have,have_now,out}, updated_at)`
- `lists(id, user_id, name, created_at)` · `list_items(list_id, ingredient_id NULLABLE, raw_text, qty_json /*merged quantities*/, recipe_ids[], checked, aisle, sort)`
- `week_picks(user_id, recipe_id, added_at)`

## 8. Feature specs

**F1 — Capture & clean.** Share/paste URL → extraction ladder; photos → vision path; result shows a clean card + "source" link; low-confidence fields (yellow) editable inline; duplicates detected (same URL or ≥ 0.9 title+ingredient similarity) with merge prompt. *AC:* fixture corpus of 40 sources (top recipe sites, JS-heavy sites, blogs, screenshots, handwritten card photo, TikTok caption): ≥ 95% produce a complete valid card (title, ingredients w/ quantities, ordered steps); JSON-LD path used whenever present (cost test proves no LLM call); paywalled/failed pages degrade to "paste the text" flow.

**F2 — Ingredient parsing & scaling.** Line parser handles ranges ("2–3 cloves"), unicode fractions, "to taste", parenthetical prep; scaling multiplies sanely (keeps "to taste", rounds to kitchen-real fractions, converts when units get awkward: 16 tbsp → 1 cup); metric⇄imperial via density table for top ~100 ingredients else volume/weight passthrough. *AC:* parser golden suite (150 real lines); scaling property tests (scale×2 then ×0.5 ≡ identity within rounding); "made it at 7 servings" remembers last scale per recipe.

**F3 — Cook mode.** Full-screen steps, swipe/tap advance, wake-lock, ingredient amounts inline-highlighted in step text (linked from parse), timers auto-detected ("simmer 20 min" → tappable 20:00 chip, multiple concurrent, notification on done), keep-position on app switch. *AC:* wake-lock verified on Android Chrome + iOS Safari PWA; concurrent timers fire correctly backgrounded (or degrade to notification-scheduled with documented platform limits).

**F4 — Pantry & cookable-now.** Onboarding staples checklist (60 s); "have now" quick toggles from any recipe or list; shelf ranks: cookable-now (100% match, staples assumed), missing-1, missing-2 (show the gap: "need: cream"). *AC:* matcher precision on fixture pantry/recipes ≥ 90% (alias handling: "scallion"="green onion"); marking a list checked-off flips pantry `have_now` for those items (opt-in setting).

**F5 — Shopping list.** Pick recipes (with per-recipe scale) → merged list: same canonical ingredient sums quantities across units where convertible, else stacked lines; aisle grouping (dict) with manual reorder learned per user; staples `always_have` auto-checked (visible, unchecked-able); share as plain text; collaborative check-off via shared link (SSE) is a stretch-goal flag. *AC:* merge suite (flour in g + cups; "1 onion" + "2 onions, diced"); checked state survives offline round-trip.

**F6 — Box & search.** Tags (auto-suggested: cuisine, meal, method), favorites, "made it" log; search across title/ingredients/tags/notes; filters (time ≤ 30 min, cookable-now, tag). *AC:* search < 100 ms locally at 1k recipes; box fully browsable offline incl. images (cached).

## 9. Milestones

- **M0 (1):** scaffold, auth, manual recipe entry, box + tags + search.
- **M1 (2):** extraction ladder (URL paths) + review-lite + duplicates. *Ships: the de-blogger.*
- **M2 (3):** ingredient parser + scaling/conversion + photo/vision capture.
- **M3 (4):** cook mode + timers + wake-lock; offline box.
- **M4 (5):** pantry + matcher + cookable-now; shopping lists + merge. *Ships: v1.0.*

## 10. Testing

- Extraction corpus as the permanent regression gate (recorded fixtures in CI; nightly live run with per-site pass/fail so site breakage is detected before users report it).
- Parser/scaler/merger: golden + property tests (the math must never embarrass us).
- Playwright mobile: share-URL → card → cook mode → timer; offline box browse; list check-off.
- Dict CI checks: alias uniqueness, aisle coverage.

## 11. Risks & open questions

- Site scraping fragility/ToS → we snapshot for personal use with source attribution and always keep the paste-text fallback; no republishing (private box), which keeps this in personal-archival territory.
- Ingredient canonicalization long tail → dict + embeddings fallback, and unresolved items just behave as raw lines (graceful degradation everywhere).
- LLM cost per capture → ladder ensures JSON-LD (a huge share of recipe sites) costs $0; target average < $0.005/capture.
