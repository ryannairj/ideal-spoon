# Implementation Spec — Warranty Vault

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Keepsafe Receipts** (app id `warrantyvault`) · proprietary; self-host compose shipped |
| Stack | Next.js 15 PWA + TS 5; Postgres 16 (Drizzle); Redis + BullMQ; S3-compatible; Auth.js magic link; Resend (email out) + Postmark inbound (email-in) — decision: **Resend for outbound, Postmark for inbound-parse** |
| Extraction model | `claude-haiku-4-5` vision; retry once w/ error feedback; `tesseract.js` server-side fallback populates raw_text only |
| Extraction schema (zod, fixed) | `{merchant, purchased_at, currency, total_cents, payment_hint, items:[{name, qty, unit_price_cents, category, serial?}], confidence: {merchant, date, total, items}}` categories from fixed list: `electronics, appliance, furniture, clothing, tool, toy, sports, jewelry, garden, auto, grocery, service, other` |
| Knowledge pack | `packs/{au.yaml, us.yaml, eu.yaml, uk.yaml}` in-repo, versioned; entries: merchants `{name, aliases[], return_days, holiday_extension?, url}` + category defaults `{category → warranty_months}` + statutory notes per region |
| Warranty source precedence | user override > merchant policy > category default; provenance stored + displayed |
| Deadlines | return_deadline = purchased_at + return_days (merchant/user); reminders return: T−3d, T−1d; warranty: T−30d, T−7d; tz = household tz |
| Email-in | per-household `v-<nanoid10>@in.<domain>`; allowlist = member emails + confirmed extras; non-allowlisted → quarantine (visible list, 14 d retention per row — see §9) |
| Multi-item split | every extracted item ≥ $10 (setting `min_split_cents=1000`) becomes its own product; below → grouped into one "receipt items" product |
| Dossier PDF | server-side via `@react-pdf/renderer` |
| Freemium | 30 active products free; Stripe subscription unlocks unlimited (flag `BILLING=off` for self-host) |
| Ports | web 3000, worker 3008 |

## 1. Repository layout

```
warrantyvault/
  src/
    app/
      (app)/{home/page.tsx (countdown rails), capture/page.tsx, receipt/[id]/confirm/page.tsx,
             product/[id]/page.tsx, library/page.tsx, quarantine/page.tsx, household/page.tsx,
             settings/page.tsx}
      api/{upload/route.ts, inbound-email/route.ts (postmark webhook), receipts/…, products/…,
           reminders/…, export/route.ts, dossier/[id]/route.ts, push/…, stripe/webhook/route.ts}
    services/{extract.ts, knowledge.ts (pack loader + join), deadlines.ts, remindEngine.ts,
              emailIngest.ts, splitter.ts, dossier.tsx}
    components/{CaptureButton, ConfirmScreen (one-scroll, low-confidence highlights),
                CountdownCard, ProductRow, ClaimChecklist, DocAttach, MemberList,
                QuarantineRow, SearchBar, LifecycleActions (returned/claimed/disposed)}
    lib/outbox.ts
  worker/src/{index.ts, jobs/{extract.ts, remind.ts, retention.ts}}
  packs/*.yaml  packs/schema.ts (zod + CI validation)
  packages/db/schema.ts
  fixtures/receipts/ (60 labeled: photos + emails .eml)  fixtures/llm/
  docker-compose.yml
```

## 2. Dependencies

`next, react, drizzle-orm, postgres, bullmq, ioredis, @aws-sdk/client-s3, sharp, zod, yaml, idb, web-push, resend, postmark (inbound verify), stripe, @react-pdf/renderer, tesseract.js, date-fns-tz, tailwindcss, mailparser (eml fixtures + attachment extraction)`. Dev: vitest, playwright.

## 3. Database schema (concretions)

PLAN §7 tables with: `receipts.extracted jsonb` (raw model output + confidences), `receipts.status` also `'quarantined'`; `products.warranty_source` enum + `products.provenance jsonb {return: source, warranty: source, pack_version}`; `products.status` transitions guarded in service (`active → returned|claimed|disposed|expired`; reminders cancelled on any exit from active — trigger-free, service + test). Add:
```sql
CREATE TABLE email_allowlist (household_id uuid, email text, confirmed boolean, PRIMARY KEY(household_id, email));
CREATE TABLE quarantine (id uuid PK, household_id uuid, from_email text, subject text,
  blob_refs jsonb, received_at timestamptz, expires_at timestamptz);
CREATE INDEX products_deadlines ON products(household_id, return_deadline, warranty_until);
products.tsv tsvector GENERATED (name, notes, category) — plus receipts.raw_text FTS index; search joins both.
```

## 4. API contract (route handlers; all household-scoped via membership)

`POST /api/upload` (photo/pdf multipart, outbox idempotency) → receipt draft + job · `POST /api/inbound-email` (postmark signature verify → emailIngest) · `GET /api/receipts/{id}` / `POST /api/receipts/{id}/confirm {edits}` → splitter → products + deadlines + reminders · `PATCH /api/products/{id}` (fields, serial, notes, status w/ guard) · `POST /api/products/{id}/docs` (manuals/warranty cards) · `GET /api/products?filter=closing|active-warranty|category:… &q=` · `POST /api/reminders/{id}/resolve {action: keep|returning|extend|claim_started}` (sibling-cancel logic) · `GET /api/dossier/{productId}` (PDF stream) · `GET /api/export` (zip) · quarantine list/approve/discard · household invite/remove (removal revokes allowlist — test).

## 5. Pipelines

**extract.ts (worker):** blob → sharp normalize (rotate/exif, max 2200 px) → vision → zod (retry 1) → knowledge.ts join: merchant match (normalized name vs aliases, fuzzy ≥ 0.85 trigram) → return_days + category warranty months + provenance → receipt draft `status='draft'` → push/SSE "ready to confirm". Failure path: tesseract raw_text + manual-entry form, capture NEVER fails.
**emailIngest.ts:** verify sig → allowlist check (else quarantine row) → HTML order? extract inline (vision on rendered-to-image? NO — decision: parse HTML text + attachments; body text goes to a text-mode extraction prompt, attachments to vision) → same downstream.
**remindEngine.ts:** on product create/update materialize reminder rows per §0 schedule (skip past ones); worker scans due (1-min tick), sends push+email fallback, quiet hours 21:00–08:00 household tz; resolve actions: `keep` cancels return siblings; `returning` keeps T−1d + adds day-of; `extend` prompts new date; warranty `claim_started` → ClaimChecklist state on product.

## 6. Screens

Home: "Closing soon" rail (CountdownCard: product, days-left badge color-coded, primary action), warranty-expiring rail, recent adds. Capture: camera/upload + "or forward emails to v-…@…" copy-address card. Confirm: one scroll — merchant/date/total header (low-confidence fields amber + tap-to-edit), item list with per-item keep/split toggles, save. Product: photos, receipt link, countdowns, provenance chips ("14-day return — JB Hi-Fi policy v2026.05, verify before relying"), serial add, docs, LifecycleActions, ClaimChecklist (gather list from pack: receipt, serial, fault description, merchant contact). Library: filters + search. Household: members, invite, email allowlist, quarantine link. Settings: region pack, retention, export, billing.

## 7. Milestone task lists

**M0** — T1 scaffold + auth/households + schema + compose; T2 manual product entry + countdown home + deadlines.ts (fake-clock tests); T3 blob upload + outbox; T4 CI + pack schema validation job.
**M1** — T1 extract worker + confirm flow + splitter (min_split rule tests); T2 packs au+us (top 50 merchants each, researched during implementation — cite URLs in pack entries) + knowledge join + provenance chips; T3 fixture corpus regression (60 receipts: merchant/date/total ≥ 90%, item split ≥ 80% on recorded fixtures; nightly live). *The magic ships.*
**M2** — T1 remindEngine full (schedules, resolve semantics, sibling cancellation, quiet hours, status-exit cancellation — all fake-clock release-blocking); T2 push + email fallback templates; T3 LifecycleActions + ClaimChecklist.
**M3** — T1 Postmark inbound + emailIngest + text-mode extraction prompt + attachment path; T2 allowlist + quarantine UI + never-silent-drop test; T3 .eml fixture suite (gmail/outlook/apple forwards, amazon multi-item, base64 attachments).
**M4** — T1 search (FTS across products+receipts raw_text) + filters + 1k-product perf; T2 household sharing polish + revocation tests; T3 export zip + import + dossier PDF (image + pdf receipt rendering test); T4 Stripe freemium (30-active limit, grandfathering rule: limit applies to `active` only) + billing UI; T5 eu/uk packs + statutory notes review + self-host docs. Tag v1.0.

## 8. Test mapping

Receipt corpus = extraction gate. Reminder lifecycle fake-clock suite = release gate (a reminder firing for a returned product is the cardinal bug). Pack CI: schema + every merchant has region+return_days+url. Playwright: capture → confirm → time-travel (test clock endpoint, dev-only) → reminder → mark returned.

## 9. Normalization, dossier format & Error Recovery

**Merchant-name normalization (`knowledge.ts`).** Before fuzzy matching against pack aliases, both the extracted merchant and every alias pass through the same normalizer so matching is stable:

```
function normMerchant(s):
  s = lowercase(s)
  s = strip diacritics (NFKD → drop combining marks)
  s = remove legal suffixes: {pty ltd, ltd, inc, llc, gmbh, co, corp, plc}
  s = remove punctuation, collapse whitespace
  s = drop leading "the "
  return s
# match: exact on normalized → else trigram ≥ 0.85 on normalized → else unmatched (category default only)
```

AC: "JB Hi-Fi Pty Ltd", "jb hi fi", and "JB HiFi" all resolve to the same pack entry.

**Receipt embedding in PDF dossier (`dossier.tsx`).** The dossier embeds the receipt as an **image page**, not raw bytes: PDFs and photos are first rasterized to PNG (max 1600 px long edge) and placed on a full page via `@react-pdf/renderer` `<Image>`; original file is also attached as a linked appendix reference (filename + S3 key note). Multi-page source → one dossier page per source page. This guarantees the dossier renders identically regardless of source MIME. AC: dossier test renders for both a `.jpg` and a multi-page `.pdf` receipt.

**Quarantine timeout scope.** Retention is **per quarantine row** (per inbound email), not per household: each row's `expires_at = received_at + 14 d`; the retention job hard-deletes individual expired rows + their blobs. Approving a row moves it out of quarantine (its timer no longer applies). AC: two emails received a day apart expire a day apart.

**Multi-item split stickiness on price edit.** During confirm, each item has an explicit `split: keep|group` toggle. Once the user toggles an item, that choice is **sticky** and the `min_split_cents` heuristic no longer overrides it — even if a later price edit crosses the $10 threshold. Items the user never touched continue to follow the live threshold. Persisted as `item.splitOverride: bool`. AC: user groups a $12 item, then edits its price to $40 → it stays grouped.

**Reminder cancellation cascade (duplicate products).** When two products are merged or one is deleted, `remindEngine` cancels the losing product's reminders atomically in the same transaction as the status change (no orphan reminders). Duplicate-product case: if the same receipt is confirmed twice (idempotency miss), the second confirm detects an existing product with identical `(household, merchant, purchased_at, total_cents)` and reuses it rather than creating duplicate reminders. AC (new test): re-confirming a receipt does not double reminders; deleting a product leaves zero due reminders for it.

**Error Recovery & Graceful Degradation:**

| Failure | Trigger | Backoff / handling | Fallback | User-facing UX |
|---|---|---|---|---|
| Vision extraction invalid | zod fails on model output | 1 re-prompt with error list | `tesseract.js` populates `raw_text` → manual-entry confirm form | "couldn't auto-read — fill in the details (we saved your receipt)"; capture never fails |
| Inbound email parse fails | mailparser error or no usable body/attachment | — | store as quarantine row with raw `.eml` blob | quarantine row flagged "couldn't parse — open manually" |
| Reminder send fails | push + email both error | retry 3× base 5 s ×2 cap 60 s | reminder stays `due`, re-attempted next tick; not marked sent | home rail still shows the countdown (never silently lost) |
| Pack load/validate fails | malformed `packs/*.yaml` at boot | CI blocks; at runtime skip bad pack, load others | category defaults only for that region | admin log warns; provenance shows "category default" |
| Stripe webhook out of order | events arrive late/duplicated | idempotent by event id; reconcile from subscription status | entitlement recomputed from current sub state | none |
