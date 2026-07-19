# Implementation Spec — Unrecipe

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Unrecipe** · proprietary; self-host compose shipped |
| Stack | Next.js 15 PWA + TS 5; Postgres 16 + pgvector (Drizzle); Redis + BullMQ; S3 (snapshots/images); Auth.js magic link |
| Extraction ladder | (1) JSON-LD/microdata `schema.org/Recipe` (`extract/jsonld.ts`, no LLM); (2) `@mozilla/readability` DOM → `deepseek-chat` normalize; (3) photos/screenshots → `claude-haiku-4-5` vision. All → recipe zod schema |
| Recipe schema (zod, fixed) | `{title, servings, total_min?, image?, ingredients:[{raw, qty?, qty_hi?, unit?, item, prep?, optional: bool}], steps:[{text, timer_sec?}], tags[], source:{kind, url?, snapshot_key}}` |
| Ingredient parser | deterministic TS (`core/lineparse.ts`): unicode fractions, ranges, "to taste", parentheticals; LLM never parses lines (only whole-page normalization which must preserve raw lines) |
| Unit conversion | kitchen table in `core/units.ts`; density table top 100 ingredients (`content/densities.yaml`); rounding to kitchen fractions {⅛,¼,⅓,½,⅔,¾} |
| Canonical dict | `content/ingredients.yaml` ~800 entries `{id, canonical, aliases[], aisle, staple: bool}`; unresolved → embedding match ≥ 0.85 else raw-line passthrough |
| Timer detection | regex on steps: `(\d+)[-–]?(\d+)?\s*(min|minute|hour|hr)s?` → timer_sec (range → upper) |
| Dup detection | same URL OR (title trigram ≥ 0.7 AND ingredient-set Jaccard ≥ 0.6) → merge prompt |
| Cookable-now | staples assumed present; rank groups: missing-0, missing-1, missing-2 with gap list |
| List merge | same canonical id: sum when units convertible (via density if cross volume/weight), else stacked sub-lines |
| Offline | recipe box + images cached (service worker + IndexedDB mirror of user's recipes); capture queues |
| Ports | web 3000, worker 3009 |

## 1. Repository layout

```
unrecipe/
  src/
    app/
      (app)/{box/page.tsx, r/[id]/page.tsx, r/[id]/cook/page.tsx, add/page.tsx,
             pantry/page.tsx, shelf/page.tsx (cookable-now), lists/page.tsx, list/[id]/page.tsx,
             week/page.tsx, settings/page.tsx}
      share/page.tsx (share-target)  api/{capture/route.ts, recipes/…, pantry/…, lists/…, sse/…}
    components/{RecipeCard, IngredientTable (scale-aware), StepList, CookMode (wake-lock, timers),
                TimerChip, ScaleControl, UnitToggle, PantryToggleRow, ShelfSection,
                ListView (aisle groups, check-off), PickTray (week picks), DupMergeDialog,
                ConfidenceField, TagChips}
    core/{lineparse.ts, scale.ts, units.ts, canon.ts (dict+embeddings), merge.ts, timers.ts}
    lib/{offline.ts (idb mirror + sw), outbox.ts}
  worker/src/{index.ts, jobs/{capture.ts (ladder), embed.ts}}
  content/{ingredients.yaml, densities.yaml}  packages/db/schema.ts
  fixtures/{sources/ (40: html snapshots + photos), lines.yaml (150 parser cases), merge.yaml}
  docker-compose.yml
```

## 2. Dependencies

`next, react, drizzle-orm, postgres, bullmq, ioredis, @aws-sdk/client-s3, sharp, zod, yaml, @mozilla/readability, jsdom (server-side readability), idb, web-push, minisearch (client search), date-fns, tailwindcss, fraction.js`. Dev: vitest, fast-check, playwright.

## 3. Database schema (key DDL)

```sql
CREATE TABLE recipes (id uuid PK, user_id uuid NOT NULL, title text, source_kind text,
  source_url text, snapshot_key text, image_key text, servings int, total_min int,
  ingredients jsonb NOT NULL, steps jsonb NOT NULL, tags text[], note_md text,
  made_count int DEFAULT 0, last_scale real, favorite boolean DEFAULT false,
  confidence jsonb, embedding vector(1536), created_at timestamptz,
  tsv tsvector GENERATED (title + ingredients item text + tags + note));
CREATE TABLE ingredient_dict (id text PK, canonical text, aliases text[], aisle text,
  is_staple boolean, embedding vector(1536));      -- seeded from content/ingredients.yaml
CREATE TABLE pantry (user_id uuid, ingredient_id text, state text CHECK (state IN
  ('always_have','have_now','out')), updated_at timestamptz, PRIMARY KEY(user_id, ingredient_id));
CREATE TABLE lists (id uuid PK, user_id uuid, name text, created_at timestamptz);
CREATE TABLE list_items (id uuid PK, list_id uuid, ingredient_id text, raw_text text,
  qty jsonb, recipe_ids uuid[], checked boolean DEFAULT false, aisle text, sort text);
CREATE TABLE week_picks (user_id uuid, recipe_id uuid, scale real DEFAULT 1, added_at timestamptz,
  PRIMARY KEY(user_id, recipe_id));
```

## 4. API contract

`POST /api/capture {url | text | photo multipart}` → job → SSE `recipe_ready {id, needsReview: fields[]}` · `GET/PATCH/DELETE /api/recipes/{id}` (PATCH incl. inline field fixes, tags, note, made-it++ with scale) · `GET /api/recipes?filter=cookable|time30|tag&q=` · `POST /api/recipes/{id}/merge {dupId}` · pantry: `PUT /api/pantry {ingredientId, state}` bulk · `POST /api/lists {recipeIds: [{id, scale}]}` → merged list · `PATCH /api/list-items/{id} {checked}` (+ optional pantry flip per setting) · week picks CRUD · `GET /api/export`.

## 5. Core algorithm contracts

**capture.ts ladder:** URL → fetch (UA rotation, 10 s) → snapshot to S3 → jsonld.ts hit? (validate against zod, fill gaps like timer detection) : readability → normalize prompt ("extract EXACTLY the recipe; ingredients as the original lines" → then lineparse each raw line deterministically) → confidence per field; photo → vision → same post-processing. Paywall/фfetch fail → respond `needs_paste` state. Cost test: JSON-LD path makes 0 LLM calls.
**scale.ts:** qty × factor → fraction.js → kitchen rounding; unit promotion (16 tbsp → 1 cup) when result crosses table thresholds; "to taste"/optional untouched; property: scale(2)∘scale(0.5) ≡ id within rounding ε.
**merge.ts (lists):** group by canonical id → partition by convertibility graph (volume↔volume, weight↔weight, cross via density if known) → sum per partition → render lines; unresolved raws stacked verbatim with recipe refs.
**canon.ts:** normalize (lowercase, strip prep words list) → dict alias exact → trigram ≥ 0.9 → embedding ≥ 0.85 → else null (raw passthrough). Corrections (user re-links) stored as user-scope aliases, win first.

## 6. Cook mode contract

Full-screen route; `navigator.wakeLock` (re-acquire on visibilitychange); step swipe/tap zones; ingredient amounts inline-bolded (linked from parse spans — lineparse returns char offsets into raw); TimerChips per detected timer: tap → countdown chip (multiple concurrent, Web Notification at 0; if backgrounded and notifications denied → on-return banner); position persisted per recipe.

## 7. Milestone task lists

**M0** — T1 scaffold + auth + schema + compose + dict seed script; T2 manual recipe entry + box grid + tags + minisearch; T3 offline mirror (sw + idb, box browsable airplane-mode test).
**M1** — T1 jsonld extractor + fixtures (20 of the 40 have JSON-LD); T2 readability→normalize path + confidence + review-lite UI (ConfidenceField ambers); T3 dup detection + DupMergeDialog; T4 corpus gate: ≥95% complete-valid cards on recorded fixtures; T5 share-target + paste-text fallback flow. *De-blogger ships.*
**M2** — T1 lineparse.ts vs `fixtures/lines.yaml` (150 golden) + char-offset spans; T2 scale.ts + units.ts + densities + property tests + ScaleControl/UnitToggle UI; T3 photo/vision path + screenshot fixtures; T4 last_scale memory.
**M3** — T1 CookMode complete (wake-lock verified Android/iOS PWA doc, timers, offsets bolding); T2 timer bg behavior matrix + on-return banner; T3 made-it log.
**M4** — T1 pantry (onboarding staples 60 s flow, PantryToggleRow everywhere) + canon.ts + correction aliases; T2 cookable-now shelf (missing-N sections + gap chips; alias test scallion=green onion; ≥90% matcher precision on fixture set); T3 lists (PickTray → merged ListView, aisle groups w/ learned reorder (user sort persisted per aisle), check-off → pantry flip setting, share-as-text); T4 merge suite (flour g+cups; onions counts) + offline check-off round-trip; T5 export/import + week page. Tag v1.0.

## 8. Test mapping

Extraction corpus (recorded; nightly live per-site pass/fail table published to job summary — site breakage early warning). Parser/scaler/merger golden+property suites = math gate. Playwright mobile: share URL → card → cook → timer; offline box; list flow.
