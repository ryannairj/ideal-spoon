# Implementation Spec — Plan Together

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Product name | **Huddle Up** (working title `huddleup`) |
| License | proprietary (SaaS candidate) — repo private |
| Stack | Next.js 15 app router + TS 5, tRPC 11, Drizzle + Postgres 16, Auth.js v5 (magic link via Resend + Google) |
| Workspaces | single Next.js app + `worker/` (tsx process for jobs) + `packages/db` |
| Realtime | Postgres LISTEN/NOTIFY → SSE endpoint `/api/trips/[id]/events`; optimistic updates via TanStack Query |
| Guest auth | signed JWT (`gt_` cookie per trip) carrying `{tripId, memberId}`; upgrade merges by memberId |
| Job queue | `jobs` table + worker poll (2 s) — no Redis (keep infra = 1 DB) |
| Link unfurl | server fetch with 5 s timeout, SSRF guard (rules in §10), og/oEmbed parse, cache 7 d |
| Currency | one currency per trip, set at creation; amounts in integer cents |
| Settlement | greedy min-cash-flow algorithm (max n−1 transfers) |
| ICS | `ics` npm package; property set fixed in §10; calendar invites emailed on date lock |
| IDs | UUIDv7; slugs `nanoid(10)` for invite tokens |
| Ports | web 3000, worker healthz 3002 |

## 1. Repository layout

```
huddleup/
  src/
    app/
      page.tsx                       # marketing + create trip
      t/[slug]/page.tsx              # trip home (tabs container)
      t/[slug]/{dates,proposals,itinerary,money,activity}/page.tsx
      j/[token]/page.tsx             # join flow (name-only guest)
      s/[shareToken]/page.tsx        # read-only itinerary share
      api/trpc/[trpc]/route.ts  api/trips/[id]/events/route.ts (SSE)
      api/ics/[tripId]/route.ts      # ICS feed after lock
    server/
      routers/{trip.ts, member.ts, availability.ts, proposal.ts, vote.ts,
               itinerary.ts, expense.ts, comment.ts, activity.ts}
      services/{decision.ts (window ranking), settle.ts, unfurl.ts, nudge.ts,
                merge.ts (guest→user), notify.ts}
      auth.ts guestAuth.ts
    components/
      dates/{AvailabilityGrid.tsx, HeatmapRow.tsx, BestWindows.tsx, LockDialog.tsx}
      proposals/{ProposalCard.tsx, ProposalForm.tsx, VoteBar.tsx, DeadlineChip.tsx}
      itinerary/{DayColumn.tsx, ItineraryItem.tsx, UnplacedTray.tsx}
      money/{ExpenseForm.tsx, BalanceList.tsx, SettleUpSheet.tsx}
      shared/{MemberAvatars.tsx, CommentThread.tsx, NeedsActionRail.tsx, EmojiPicker.tsx}
  worker/src/{index.ts, jobs/{deadlineClose.ts, nudge.ts, digest.ts, unfurlRetry.ts}}
  packages/db/src/{schema.ts, migrations/}
  e2e/ (playwright)
```

## 2. Dependencies

`next`, `react`, `@trpc/server|client|react-query`, `@tanstack/react-query`, `drizzle-orm`, `postgres`, `next-auth@5`, `resend`, `zod`, `ics`, `nanoid`, `jose` (guest JWT), `open-graph-scraper` (wrapped by ssrf guard using `ipaddr.js`), `@dnd-kit/core` (itinerary drag), `date-fns`, `tailwindcss@4`, `web-push`. Dev: vitest, playwright, `fast-check` (property tests).

## 3. Configuration

`DATABASE_URL, AUTH_SECRET, AUTH_RESEND_KEY, GOOGLE_CLIENT_ID/SECRET, GUEST_JWT_SECRET, VAPID_PUBLIC/PRIVATE, APP_URL`. Trip settings defaults: `vote_threshold: 0.5`, `nudge_schedule: [24h, 2h]`, `digest_day: 'sunday'`.

## 4. Database schema (delta-level detail; Drizzle source of truth)

Tables exactly as PLAN §7 with these concretions: all `*_id` uuid FKs `ON DELETE CASCADE` from trips; `members.guest_token_hash` sha256; `availability` PK `(trip_id, member_id, date)`; `votes` PK `(proposal_id, member_id)`; `proposals.vote_deadline timestamptz`; `itinerary_items.sort` = fractional index (string, `a0`-style keys via `fractional-indexing` pkg); `expenses.split_json` = `[{memberId, shareCents}]` must sum to amount (DB check via trigger is skipped; enforced in service + test); `activity` append-only, notify channel `trip_<id>` payload `{activityId}`; `jobs(id, kind, run_at, payload jsonb, locked_by, done_at)` index on `(done_at, run_at)`.

## 5. tRPC surface (procedures; all trip-scoped ones authorize member-of-trip)

`trip.create/get/updateSettings/lockDates/share` · `member.joinByToken({token, name})→{guestJwt}` / `member.upgrade` (post-login merge) / `member.list/remove` · `availability.setRange({dates: [{date,state}]})` / `availability.grid` · `proposal.create({kind,title,url?,estCost?,deadline})/list/lock/reject` · `vote.cast({proposalId, value})` · `itinerary.addItem/move({id, day, beforeSortKey})/update/remove` · `expense.add/list/balances/settlePlan/markSettled` · `comment.add/list({target,targetId})` · `activity.feed({cursor})`. SSE event types: `activity`, `vote_tally`, `itinerary_change`, `presence` (optional later).

## 6. Core algorithms

**Best windows (`decision.ts`):** input organizer target nights `N`, candidate range. For every window of length N and N±1: score = `2×(#yes-per-day summed) − 3×(#no) + 0.5×(#maybe)`, normalized by member count; return top 5 with per-member breakdown; ties → earlier start. Property tests: adding a `yes` never lowers that window's rank; window with any member fully `no` flagged (not hidden).

**Settlement (`settle.ts`):** balances from expenses (payer +amount, splits −share); greedy: repeatedly match max creditor with max debtor, transfer min(|amounts|); assert transfers ≤ n−1 and sum-to-zero (fast-check).

**Deadline close (`deadlineClose.ts`):** for each due proposal, in one tx and in this exact order: (1) set `voting_closed = true` (writes are now rejected by `vote.cast` — see below); (2) compute the final tally over votes with `created_at <= vote_deadline` (votes are timestamped; a vote racing the exact deadline instant is excluded — deadline is exclusive); (3) if up-share > `vote_threshold` → append activity `suggest_lock` (organizer CTA), else `deadline_passed`. Ordering rationale: disabling first guarantees the tally is stable and no vote can land between tally and close. `vote.cast` re-checks `voting_closed` inside its own tx and returns `PRECONDITION_FAILED` if set (idempotent-safe). Nudges: schedule rows created at proposal create per settings, cancelled on that member's vote.

**Guest merge (`merge.ts`):** in one tx: repoint `members.user_id`, keep memberId (votes/expenses/comments untouched — they key on memberId), delete guest token. Test: full-object graph diff before/after.

## 7. Screens & key components

Trip home = header (title, dates state, MemberAvatars, share button) + NeedsActionRail + tab nav. `dates`: AvailabilityGrid (tap/drag paint yes/maybe/no on mobile; per-day heat column), BestWindows list → LockDialog (sends ICS). `proposals`: filter chips by kind/status; ProposalCard (unfurl preview img/title/price, VoteBar avatars, DeadlineChip countdown, comments). `itinerary`: horizontal day columns (locked range), dnd-kit drag, UnplacedTray of locked proposals. `money`: expense list, BalanceList (net per member), SettleUpSheet (plan + mark paid). `activity`: feed. Join page: name input → instant membership. Share page: read-only itinerary + dates.

## 8. Milestone task lists

**M0** — T1 scaffold + schema + Auth.js (magic link, Google); T2 trip create + slugs + invite token page + guest JWT join; T3 member list/roles; T4 CI + preview deploy config (Vercel-compatible but standard Node build).
**M1** — T1 availability grid (paint UX, mobile-first) + upserts; T2 decision.ts + BestWindows + property tests; T3 LockDialog → status locked + ICS email + ICS feed route; T4 SSE channel + activity plumbing (first event types). *Ships better-than-Doodle.*
**M2** — T1 proposals CRUD + unfurl service (SSRF guard + tests: private-IP literal, DNS-rebind-style redirect, 30-redirect chain) + cache; T2 voting + tallies + live updates; T3 worker + jobs table + deadlineClose + nudges (fake-clock tests); T4 guest-merge flow + tests.
**M3** — T1 itinerary (dnd-kit, fractional index, day columns) + sync (<2 s via SSE); T2 UnplacedTray from locked proposals; T3 comments on proposals/items/expenses; T4 NeedsActionRail (unvoted + unfilled availability queries); T5 read-only share route.
**M4** — T1 expenses + balances + settle + markSettled; T2 weekly digest job + emails (only-if-activity test); T3 PWA (manifest, push for nudges/deadlines), quiet hours; T4 e2e happy path (create → 3 guests → lock → propose → vote → itinerary → expense → settle) desktop + mobile viewport. Tag v1.0.

## 9. Test mapping

fast-check: windows + settlement invariants. Fake-clock worker suite (deadlines/nudges/digests fire within 1 min). SSRF guard release-blocking. Playwright E2E incl. guest→account upgrade preserving votes. ICS snapshot test imported-validated with `node-ical`.

## 10. ICS property set & SSRF guard rules

**ICS emission (`itinerary.ts` lock path + `/api/ics/[tripId]` feed).** One `VEVENT` per locked trip (feed adds one per locked itinerary item later). Fixed property set:

| Property | Value |
|---|---|
| `UID` | `<tripId>@huddleup` (feed items: `<tripId>-<itemId>@huddleup`) — stable across re-emits so updates replace, not duplicate |
| `DTSTAMP` | emit time, UTC (`…Z`) |
| `DTSTART`/`DTEND` | trip = all-day `VALUE=DATE` (`YYYYMMDD`); `DTEND` = last night + 1 day (ICS end-exclusive). Timed feed items use `TZID=<trip.timezone>` with local `YYYYMMDDTHHMMSS` |
| `SEQUENCE` | bumped on every re-lock/edit so calendars refresh |
| `SUMMARY`/`LOCATION`/`URL`/`DESCRIPTION` | trip title / destination / share URL / member list |
| `ORGANIZER` | organizer email (invite email path only) |

Timezone: trip carries an IANA `timezone`; a single matching `VTIMEZONE` block is emitted whenever any timed component is present. All-day trip event needs no `VTIMEZONE`. Snapshot test round-trips through `node-ical` and asserts UID stability + end-exclusive DTEND.

**SSRF guard (`unfurl.ts`, wraps `open-graph-scraper`).** Applied on the initial URL and re-applied after every redirect hop:

1. Scheme must be `http`/`https`; reject others (`file:`, `gopher:`, etc.).
2. Resolve host DNS; reject if **any** resolved A/AAAA is private/reserved via `ipaddr.js` range check: `private`, `loopback`, `linkLocal`, `uniqueLocal`, `carrierGradeNat`, `reserved`, `unspecified`, plus literal IPv4-mapped IPv6. Pin the vetted IP and connect to it (custom `lookup`) so DNS cannot rebind between check and fetch.
3. Reject non-standard ports (allow 80/443 only).
4. Follow ≤ 5 redirects; re-run steps 1–3 on each `Location`; a redirect to a private target fails the whole fetch.
5. Total wall-clock ≤ 5 s, response body cap 2 MiB (stream-abort past cap).

On any rejection: no preview stored, `unfurl_status = 'blocked'`, card shows the bare link. Tests (release-blocking): private-IP literal, DNS-rebind-style redirect to `169.254.169.254`, 30-redirect chain, non-HTTP scheme, oversized body.
