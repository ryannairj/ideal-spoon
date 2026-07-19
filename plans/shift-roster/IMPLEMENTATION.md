# Implementation Spec — ShiftReady (rostering & timesheets)

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **ShiftReady** · proprietary |
| Stack | Next.js 15 + TS 5 + tRPC 11; Postgres 16 + Drizzle with **RLS enforced** (session sets `app.org_id`); Auth.js v5 magic link; BullMQ + Redis worker |
| Route trees | `/manage/*` (owner/manager), `/me/*` (staff PWA), `/kiosk/*` (tablet mode) |
| Time handling | ALL times stored UTC; org.tz for display & rule evaluation; `date-fns-tz`; shifts store `date` (org-local) + `start`/`end` minutes-from-midnight + computed UTC columns |
| Overlap guard | Postgres exclusion constraint on `(membership_id, tstzrange(start_utc, end_utc))` where status != draft-unassigned |
| Rule packs (fixed rule ids) | `min_rest_h` (default 10), `max_week_h` (38), `max_consecutive_days` (6), `break_rules` `[{after_h: 5, break_min: 30}]` — warnings only, server-validated |
| Swap policy | claim → pending approval default; org setting `auto_approve_claims` |
| Clock matching | punch → shift within ±4 h window same membership, else `unmatched` exception |
| Auto-approve tolerance | actual within ±5 min of rostered → auto-approvable in bulk |
| Exports | generic CSV + Xero timesheet CSV (contract fixtures); export locks period (`timesheets.status='exported'`, immutable) |
| Billing | Stripe flat per location/month, 30-day trial |
| Geofence | soft: record `geo_ok` boolean, never block |
| Ports | web 3000, worker 3006 |

## 1. Repository layout

```
shiftready/
  src/
    app/
      manage/{roster/page.tsx, staff/page.tsx, leave/page.tsx, timesheets/page.tsx,
              exports/page.tsx, settings/{page.tsx, rules/page.tsx, billing/page.tsx},
              onboarding/page.tsx}
      me/{page.tsx (my shifts), availability/page.tsx, leave/page.tsx, open-shifts/page.tsx,
          swaps/page.tsx, clock/page.tsx}
      kiosk/page.tsx
      api/{trpc/…, stripe/webhook/route.ts, ics/[memberToken]/route.ts}
    server/
      routers/{org.ts, staff.ts, availability.ts, leave.ts, roster.ts, publish.ts,
               openshift.ts, swap.ts, clock.ts, timesheet.ts, export.ts, billing.ts}
      services/{rules.ts, eligibility.ts, publishDiff.ts, matchPunch.ts, periods.ts,
                notify.ts, xero.ts}
      rls.ts (org context middleware)
    components/
      roster/{WeekGrid.tsx (virtualized rows), ShiftBlock.tsx, ShiftEditor.tsx,
              ViolationBadge.tsx, CostBar.tsx, TemplateMenu.tsx, PublishDialog.tsx}
      me/{ShiftCard.tsx, ConfirmBar.tsx, AvailabilityEditor.tsx, ClockButton.tsx, SwapSheet.tsx}
      kiosk/{PinPad.tsx, PunchScreen.tsx}
      timesheets/{PeriodTable.tsx, VarianceQueue.tsx, ExceptionRow.tsx}
  worker/src/{index.ts, jobs/{notifyFanout.ts, missedPunch.ts, reminders.ts}}
  packages/db/{schema.ts, rls.sql, migrations/}
  fixtures/{xero-expected.csv, dst-scenarios.yaml}
  e2e/
```

## 2. Dependencies

`next, react, @trpc/*, @tanstack/react-query, drizzle-orm, postgres, next-auth@5, bullmq, ioredis, date-fns, date-fns-tz, zod, stripe, web-push, resend, @dnd-kit/core (grid drag), fractional-indexing, tailwindcss, ics`. Dev: vitest, fast-check, playwright.

## 3. Schema concretions (beyond PLAN §7)

All tables carry `org_id` + RLS policy `USING (org_id = current_setting('app.org_id')::uuid)`; `rls.sql` applied in migrations; tRPC context sets the GUC per request after membership check. Key additions:
```sql
ALTER TABLE shifts ADD COLUMN start_utc timestamptz GENERATED …; -- via trigger (org tz join) — implement as trigger fn
ALTER TABLE shifts ADD CONSTRAINT no_overlap EXCLUDE USING gist
  (membership_id WITH =, tstzrange(start_utc, end_utc) WITH &&)
  WHERE (membership_id IS NOT NULL AND status IN ('published','confirmed'));
CREATE TABLE publishes (id uuid PK, org_id uuid, location_id uuid, week date,
  diff jsonb, published_by uuid, at timestamptz);  -- snapshot for diff-vs-last
CREATE TABLE timesheet_lines (timesheet_id uuid, shift_id uuid, rostered_min int,
  actual_min int, approved_min int, variance_reason text, approved_by uuid,
  status text CHECK (status IN ('draft','approved','exception')), PRIMARY KEY(timesheet_id, shift_id));
```
`clock_events` append-only: no UPDATE/DELETE grants (enforced via revoked privileges + test).

## 4. tRPC surface (org-scoped; roles enforced per procedure)

`org.create/settings/rules.get|set` · `staff.invite/list/update/remove` (remove → revoke + flag future shifts) · `availability.setPattern/setException/get` · `leave.request/approve/deny/list` · `roster.week({locationId, week})` (staff rows, shifts, violations computed) / `roster.upsertShift` (server re-validates rules + availability + leave → violations[] in response; client renders badges) / `roster.copyWeek/templates.apply` · `publish.preview({week})→diff` / `publish.commit` (fan-out notifications to affected only) · `openshift.eligible({shiftId})` / `openshift.claim` (tx: first pending wins) / `swap.offer/claim/approve/deny` · `clock.kioskPunch({pin, kind})` / `clock.phonePunch({kind, geo})` · `timesheet.period({period})/approveLine/bulkApproveWithinTolerance/resolveException` · `export.generic({period})` / `export.xero` / both lock period · `billing.portal/status`.

## 5. Core services

**rules.ts**: pure functions `(proposedShift, memberWeekContext) → Violation[]` with ids + human messages ("Only 8 h rest after Tue close"); golden tables per rule; server authoritative, client advisory copy.
**eligibility.ts**: role match ∧ no overlap ∧ no leave ∧ under max_week_h ∧ available; property test vs brute-force checker.
**publishDiff.ts**: compare against last publish snapshot → `{new[], changed[], removed[]}` per member; zero-diff republish → no notifications (test).
**matchPunch.ts**: pair in/out events → shift per §0 window; missing out by shift_end + 4 h → worker creates exception + manager notification.
**periods.ts**: weekly/fortnightly periods from org setting; DST-safe duration math (all arithmetic in UTC instants; release-blocking `dst-scenarios.yaml` suite: spring-forward overnight shift = 7 h not 8, etc.).

## 6. Screens (key UX contracts)

`manage/roster`: WeekGrid — rows staff grouped by role + "Open shifts" row; drag-create (15-min snap), resize, move; ViolationBadge inline with explanation popover; CostBar live total (Σ hours × rate); TemplateMenu (save week as template, apply, copy last week); PublishDialog (diff summary, confirm). `me`: ShiftCard list + ConfirmBar (confirm/decline+reason), availability painter (weekly pattern + date exceptions), open shifts (claim), swaps sheet, clock page (big punch button, state-aware, geo request). `kiosk`: location-locked route with kiosk token; PinPad → PunchScreen (name, in/out/break buttons, confirmation flash). `timesheets`: PeriodTable per staff (rostered vs actual vs approved), VarianceQueue (outside tolerance), bulk-approve button. `onboarding`: business type → seeded roles/templates → invite staff → first roster.

## 7. Milestone task lists

**M0** — T1 scaffold + schema + RLS (`rls.sql` + two-org isolation test suite) + auth + org/location/staff invites; T2 role-gated route trees + tRPC context GUC; T3 CI + compose.
**M1** — T1 WeekGrid (virtualized, dnd, 30-staff 60 fps perf test) + shift CRUD + overlap constraint test; T2 rules.ts + violations pipeline + badges; T3 templates/copy-week; T4 CostBar.
**M2** — T1 publishes + diff + commit + notify worker (push/email prefs, digest batching) + zero-diff test; T2 staff confirm/decline + dashboard status; T3 availability + leave (+ approved-leave-conflicts-existing-shifts alert); T4 me/* PWA polish + ics feed per member.
**M3** — T1 open shifts (eligibility + race tx test: 2 concurrent claims → 1 winner) + auto-approve setting; T2 swaps full flow; T3 push notifications + email fallbacks everywhere (no-install path test).
**M4** — T1 kiosk (PIN hash per location, punch, optional photo capture); T2 phone punch + soft geofence; T3 matchPunch + missedPunch worker + exceptions; T4 timesheets build (periods.ts) + variance queue + bulk approve; T5 DST suite green (release gate).
**M5** — T1 exports (generic + xero.ts vs `xero-expected.csv` contract fixture; lock + byte-identical re-export test); T2 Stripe (flat plan, trial, webhook, portal) + gating; T3 onboarding wizard + seed templates (cafe, clinic, retail); T4 e2e full weekly cycle (manager desktop + staff mobile viewport). Tag v1.0.

## 8. Test mapping

Release gates: RLS isolation, DST suite, punch append-only, claim race, export reconciliation (totals = approved lines to the minute). fast-check: eligibility vs brute force, rules monotonicity. Playwright dual-role E2E.
