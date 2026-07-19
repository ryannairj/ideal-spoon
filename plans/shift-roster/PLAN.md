# ShiftReady — rostering & timesheets for small shift-based businesses

**Category:** SaaS · **Difficulty:** Medium–Hard · **Platform:** Web app (manager) + mobile PWA (staff)

## 1. Vision

Cafés, clinics, and small retail still roster in spreadsheets and text the PDF to a group chat; timesheets are re-typed into payroll monthly. ShiftReady is the whole loop for a ≤ 50-staff business: build the roster against availability and rules, publish to staff phones, capture swaps and clock-ins, and export payroll-ready timesheets. Deputy/When-I-Work exist — the wedge is **radical simplicity + honest small-business pricing** (flat, not per-seat-per-module) and a genuinely good staff PWA.

## 2. Why it can win

- Incumbents are enterprise-creeped: setup wizards, modules, per-user fees that sting at 20 staff. A focused core (roster → notify → swap → timesheet → export) with opinionated defaults wins the "we just left Excel" segment.
- Compliance guardrails as *warnings not walls*: min rest between shifts, max weekly hours, break rules — configurable rule packs, visible while building.

## 3. Users & use cases

Owner/managers of cafés, restaurants, pharmacies, gyms, small clinics; their staff.

1. Manager drags shifts on a week grid; conflicts (availability, leave, rest rules) flag live; copies last week as starting point.
2. Publish → staff get push/email; each confirms; unfilled shifts offered as open shifts staff can claim.
3. Staff: "can't do Thursday" → swap request → eligible colleagues notified → manager one-tap approves.
4. Clock in/out on the store tablet (kiosk PIN) or own phone (geofenced) → timesheet drafts vs rostered times → manager approves variances → CSV/Xero export.

## 4. MVP scope & non-goals

**MVP:** org/locations/roles, staff onboarding via invite, availability & leave requests, week-grid roster builder with rule warnings, templates/copy-week, publish + confirmations, open shifts & swaps, clock-in (kiosk + phone), timesheet approval, exports (CSV + Xero timesheet format), flat-fee billing (Stripe).
**Non-goals:** payroll calculation itself (export to payroll, never compute tax), award interpretation engines (rule packs are simple configurable constraints, explicitly not legal compliance), auto-scheduling AI (v2 — manual + templates first), multi-week optimization, native apps.

## 5. Tech stack

- **App:** Next.js 15 + TypeScript + tRPC, Postgres (Drizzle), Auth.js (email magic link; staff invited by phone/email), multi-tenant via org_id row scoping + RLS.
- **Staff PWA:** same codebase, mobile routes; Web Push; installable; kiosk mode route for tablets.
- **Workers:** BullMQ (notifications, reminders, exports); `date-fns-tz` everywhere — timezone correctness is core.
- **Billing:** Stripe (flat monthly per location).

## 6. Architecture

Single app, role-gated route trees (`/manage`, `/me`, `/kiosk`). All shift mutations run through a domain service that re-validates rules server-side and emits `events` rows (audit + notification fan-out). Notification worker delivers push/email with per-user preferences and digest batching.

## 7. Data model

- `orgs(id, name, tz, settings_json, billing_state)` · `locations(id, org_id, name, address, geofence_lat/lng/radius, kiosk_pin_hash)`
- `users(id, email, phone, name)` · `memberships(user_id, org_id, role{owner,manager,staff}, hourly_rate NULLABLE, roles_json /*job roles: barista, RN*/, status)`
- `availability(id, membership_id, weekday, start, end, kind{available,unavailable,preferred}) ` · `leave(id, membership_id, from, to, kind, status{pending,approved,denied}, note)`
- `shifts(id, org_id, location_id, role, date, start, end, break_min, membership_id NULLABLE /*null = open*/, status{draft,published,confirmed,declined}, notes)`
- `swaps(id, shift_id, from_membership, to_membership NULLABLE, status{offered,claimed,approved,denied}, at)`
- `clock_events(id, membership_id, shift_id NULLABLE, kind{in,out,break_start,break_end}, at, source{kiosk,phone}, geo_ok bool, device_meta)`
- `timesheets(id, membership_id, period, lines_json /*per shift: rostered vs actual, variance, approved_by*/, status{draft,approved,exported})`
- `rule_packs(org_id, rules_json /*min_rest_h, max_week_h, max_consecutive_days, break_rules*/)` · `events(id, org_id, actor, verb, target, payload, at)`

## 8. Feature specs

**F1 — Roster builder.** Week grid (rows = staff grouped by role, columns = days); drag-create/resize/move; unassigned row for open shifts; live cost total (hours × rates); rule + availability violations as inline badges with explanations; copy-last-week; shift templates. *AC:* all rule checks re-validated server-side (client is advisory); 30-staff week renders and drags at 60 fps; overlapping-shift creation for the same person is impossible (DB constraint + test).

**F2 — Publish & confirm.** Publish diff view ("4 new, 2 changed, 1 removed since last publish") → notifications only for affected staff; staff confirm/decline with reason; dashboard shows confirmation status. *AC:* republishing unchanged week sends zero notifications; declined shift alerts manager and offers convert-to-open.

**F3 — Availability & leave.** Staff set weekly patterns + date-specific exceptions; leave requests with manager approval; both feed builder warnings. *AC:* approved leave immediately flags conflicting existing shifts to the manager.

**F4 — Open shifts & swaps.** Open shift → eligible staff (role match, no conflict, under max hours) notified, first-claim-pending-approval or auto-approve (org setting). Swap flow per §3. *AC:* eligibility filter property-tested; race of two claimants resolves to exactly one winner (transaction test).

**F5 — Time clock.** Kiosk: PIN → punch with photo optional; Phone: geofence check (soft — records `geo_ok=false` rather than blocking). Missed-punch detection creates timesheet exceptions. *AC:* clock events are append-only; punches map to the correct shift within ±4 h window else flagged unmatched; DST transition days compute durations correctly (fake-clock tests — release-blocking).

**F6 — Timesheets & export.** Period view per staff: rostered vs actual vs approved; bulk-approve within tolerance (e.g. ±5 min auto); variance queue for the rest; export CSV (generic) + Xero timesheet CSV; export locks the period. *AC:* exported totals reconcile with approved lines to the minute; re-export of a locked period is byte-identical.

**F7 — Tenanting & billing.** RLS-enforced isolation; Stripe flat plan per location, 30-day trial, billing portal. *AC:* cross-tenant access attempts fail at the DB layer (RLS tests with two seeded orgs).

## 9. Milestones

- **M0 (1):** scaffold, multi-tenant auth + RLS, org/location/staff onboarding.
- **M1 (2):** roster builder + rules + templates (draft only). *Ships: Excel replacement.*
- **M2 (3):** publish/confirm/notifications + availability/leave. *Ships: the loop closes to staff phones.*
- **M3 (4):** open shifts + swaps; staff PWA polish.
- **M4 (5):** time clock (kiosk + phone) + timesheets + variance approval.
- **M5 (6):** exports (CSV/Xero), billing, onboarding wizard with seed templates per business type. *Ships: v1.0.*

## 10. Testing

- Timezone/DST test suite runs the whole clock→timesheet path in 3 zones incl. transition days — release-blocking.
- RLS isolation suite; race-condition tests (claims, concurrent roster edits with optimistic locking).
- Playwright: manager desktop flow + staff mobile-viewport flow end-to-end weekly cycle.
- Rule engine golden tables (rest/max-hours/breaks) per pack.

## 11. Risks & open questions

- Award/labor-law complexity is a tarpit → hard product line: configurable warnings, marketed as "guardrails, not compliance advice"; revisit only with revenue.
- Two-sided adoption (staff must install PWA) → email/SMS fallbacks for every notification; confirm links work without install.
- Xero format drift → exporter behind interface + fixture-based contract tests; add providers on demand.
