# Warranty Vault — snap a receipt, never lose a warranty or return window again

**Category:** SaaS / life improvement · **Difficulty:** Medium · **Platform:** Mobile-first PWA + email-in

## 1. Vision

Every household leaks money through expired return windows, forgotten warranties, and un-claimable receipts. Warranty Vault is the drop-box for proof-of-purchase: snap the receipt (or forward the order email), it extracts merchant/date/items/amount, attaches the right warranty length, and reminds you *before* windows close — "return period for the monitor ends Friday", "TV warranty expires next month; that flicker you mentioned? claim now."

## 2. Why it can win

- Receipt apps are expense-trackers for accountants; warranty apps are manual-entry graveyards. The wedge: **zero-entry capture** (photo/email → structured record via LLM extraction) + **deadline-centric design** (everything is a countdown, not an archive).
- Email-in (`vault@…` forwarding + Gmail label auto-forward instructions) captures online orders where most purchases actually happen.

## 3. Users & use cases

Households, gadget buyers, landlords tracking appliance warranties, small businesses (light use).

1. Buy a laptop → snap receipt at the counter → record: "JB Hi-Fi, MacBook Air, $1,799, 14-day return (Jul 26), 12-mo warranty (Jul 2027), serial: __ (add?)".
2. Forward Amazon order email → items auto-split into separate tracked products.
3. Push: "Return window for 'standing desk mat' closes in 2 days — still keeping it?"
4. Dishwasher dies at month 10 → search "dishwasher" → receipt PDF + warranty terms + claim checklist in 10 seconds.
5. Sell the house → export appliance warranty bundle as PDF.

## 4. MVP scope & non-goals

**MVP:** capture (camera, upload, email-in address), LLM extraction pipeline with confidence + one-screen confirm, product records with return/warranty countdowns, merchant knowledge pack (default return/warranty policies per known merchant + category, user-overridable), reminders (push/email), search/filters, storage of original images/PDFs, household sharing (2+ members, one vault), export (zip + CSV + per-item PDF dossier).
**Non-goals:** expense/budget analytics, claim filing automation with merchants, bank/card integrations, OCR-only offline mode (extraction needs cloud LLM; capture works offline and queues), price-drop protection tracking (v1.1 candidate), business accounting features.

## 5. Tech stack

- **App:** Next.js 15 PWA + TypeScript; offline capture outbox (IndexedDB).
- **Backend:** Postgres (Drizzle) + S3-compatible blob store; BullMQ workers.
- **Extraction:** vision LLM (receipts are wild — layout OCR alone underperforms) → structured schema with per-field confidence; `tesseract` fallback keeps raw text searchable if LLM unavailable. Email-in via inbound-parse webhook (Resend/Postmark) handling HTML orders + PDF/image attachments.
- **Knowledge pack:** versioned YAML in-repo: merchants (aliases, receipt patterns, return-days, extended-holiday rules) + category warranty defaults (electronics 12 mo, appliances 24 mo, …) per region (AU/US/EU/UK seed) including statutory-rights notes.

## 6. Architecture & pipeline

```
capture (photo/pdf/email) → blob → extract worker:
  vision LLM → {merchant, date, currency, total, payment_hint, items[{name, qty, price, category_guess, serial?}], confidence per field}
  → knowledge join: merchant/category → return_days, warranty_months (source labeled: merchant policy | category default | user)
  → draft record → user confirm screen (low-confidence fields highlighted, edit inline)
confirmed → products + deadlines materialized → reminder scheduler
```

Multi-item receipts split into one product each (shared receipt ref). Everything editable later; edits never touch the original blob (provenance preserved).

## 7. Data model

- `households(id, name)` · `users(id, email, household_id, role)` 
- `receipts(id, household_id, source{photo,upload,email}, blob_ref, raw_text, extracted_json, merchant_id NULLABLE, purchased_at, total_cents, currency, status{draft,confirmed}, created_by)`
- `products(id, household_id, receipt_id, name, category, serial, notes, price_cents, return_deadline, warranty_until, warranty_source{merchant,category,user,manual_doc}, extra_docs_json /*warranty cards, manuals*/, status{active,returned,claimed,expired,disposed})`
- `merchants(id, name, aliases[], region, return_days, notes, source_version)`
- `reminders(id, product_id, kind{return_closing,warranty_expiring,custom}, due_at, sent_at, resolved_action)`
- `events(id, household_id, actor, verb, target, at)` — shared-vault audit.

## 8. Feature specs

**F1 — Capture & extraction.** As pipeline; confirm screen is one scroll, big Save. *AC:* labeled fixture set (60 receipts: crumpled, thermal-faded, long grocery, non-English, email orders incl. Amazon multi-item): merchant/date/total each ≥ 90% exact; item split correct on ≥ 80%; every field editable; failed extraction still saves blob + raw text with manual-entry form (capture never fails).

**F2 — Email-in.** Per-household address; sender allowlist (members' emails + confirmed extras); attachments + HTML body both processed; unknown sender → quarantine list. *AC:* forwarded Gmail order lands as draft < 2 min; attachment-only email (photographed receipt) works; quarantine never silently drops (visible in UI).

**F3 — Deadlines & reminders.** Return: reminders at 3 d + 1 d before with "Keep / Returning / Extend" actions; warranty: 30 d + 7 d before expiry with "claim checklist" (what to gather, merchant contact from pack); custom reminders per product. Quiet hours; email fallback when push unavailable. *AC:* fake-clock suite covers tz + DST; resolving a reminder ("Keep") cancels its siblings; no reminder ever fires for `returned/disposed` products.

**F4 — Library & search.** Countdown-sorted home ("closing soon" rail up top); filters (category, merchant, active-warranty-only); full-text search over names/raw_text/notes; product page: photos, receipt, docs, timeline, actions (add serial, attach manual PDF, mark claimed with outcome note). *AC:* search hits receipt body text ("that thing I bought with the blue logo" territory via merchant/aliases); 1k products stay snappy (indexed queries, virtualized lists).

**F5 — Household sharing.** Invite by email; everything shared within household; per-event attribution. *AC:* member removal revokes access immediately incl. email-in allowlist.

**F6 — Export & dossiers.** Full zip (blobs + JSON + CSV); per-product PDF dossier (receipt image, key fields, warranty terms) for claims/resale. *AC:* dossier PDF renders correctly for image + PDF receipts; export restores via import losslessly.

## 9. Milestones

- **M0 (1):** scaffold, auth/households, manual product entry + blob storage + countdown home. 
- **M1 (2):** photo extraction pipeline + confirm flow + knowledge pack v1 (top 50 AU/US merchants + category defaults). *Ships: the magic.*
- **M2 (3):** reminders end-to-end + product lifecycle actions. *Ships: the value loop closes.*
- **M3 (4):** email-in + multi-item splitting + quarantine.
- **M4 (5):** search/filters polish, sharing, exports/dossiers, billing stub (freemium: 30 products free). *Ships: v1.0.*

## 10. Testing

- The 60-receipt labeled corpus is the extraction regression gate (recorded LLM fixtures in CI; nightly live with accuracy + cost report).
- Knowledge-pack schema validation + "every merchant has region+return_days" checks in CI.
- Fake-clock reminder suites; email-in webhook fixtures (Gmail/Outlook/Apple Mail forwards, base64 attachments).
- E2E: capture → confirm → time-travel → reminder → mark returned.

## 11. Risks & open questions

- Merchant policy data goes stale → policies carry `source_version` + "verify before relying" microcopy; user overrides win; community-editable pack post-v1.
- Legal-ish territory (statutory rights vary) → informational framing only, region notes cite official sources, no advice claims.
- Receipt privacy → blobs encrypted at rest, LLM data-flow disclosed, self-host compose target for the privacy-conscious.
