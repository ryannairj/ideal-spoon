# Plan Together — trips & events with friends, minus the group-chat chaos

**Category:** Social · **Difficulty:** Medium–Hard · **Platform:** Web app (mobile-first PWA)

## 1. Vision

Every group trip dies the same death: 400 WhatsApp messages, three half-polls, one person who never answers, and a spreadsheet nobody updates. Plan Together is a shared space per trip/event: decide dates, lock the plan, build the itinerary, split costs — with an opinionated flow that **drives the group to decisions** instead of hosting infinite discussion.

## 2. Why it can win

- Existing tools are either heavyweight travel products (TripIt = solo logistics) or generic polls (Doodle = one decision, no continuity). The wedge is the **decision funnel**: propose → vote with deadlines → auto-lock → itinerary — plus zero-friction join (magic link, no app install, no account wall for voting).
- Guests can participate with just a name; only organizers need accounts. This kills the #1 adoption blocker.

## 3. Users & use cases

Friend groups (4–15 people) planning trips, bachelor(ette)s, reunions, recurring dinners.

1. Create "Portugal 2027" → share one link in the group chat.
2. Everyone marks date availability (calendar heatmap) → best window auto-surfaces → organizer locks dates with one tap.
3. Anyone proposes stays/activities (paste an Airbnb/Maps link → auto-preview); votes with deadline; locked items flow into the itinerary.
4. Log costs against the trip; running "who owes whom" with simplified settlement.
5. Day-of: itinerary view with times, addresses, map links; RSVP status per event.

## 4. MVP scope & non-goals

**MVP:** trips/events, magic-link membership, date matcher, proposals + deadline voting, itinerary builder, expense splitting, comments per item, email/push nudges.
**Non-goals:** bookings/payments processing (link out; record manually), chat (comments on items only — the group already has a chat app), native apps (PWA), flight/hotel API integrations, multi-currency settlement beyond simple FX-note.

## 5. Tech stack

- **App:** Next.js 15 (app router) + TypeScript, tRPC, Postgres (Drizzle), Auth.js (magic link + Google) for organizers; signed guest tokens for participants.
- **Realtime:** Postgres LISTEN/NOTIFY → SSE (no separate infra); optimistic UI.
- **Link previews:** server-side oEmbed/OpenGraph fetcher with cache.
- **Notifs:** email (Resend) + Web Push; scheduled nudges via `pg_cron`-style queue table + worker.

## 6. Architecture

Standard three-tier: Next.js app (server components + tRPC), worker process (nudges, link unfurl, digest emails), Postgres. Every mutation writes an `activity` row → feeds SSE updates + the trip's "what changed" digest.

## 7. Data model

- `users(id, email, name, avatar)` · `trips(id, title, emoji, dates_locked_range, status{planning,locked,active,done}, settings_json, created_by)`
- `members(trip_id, user_id NULLABLE, guest_name, guest_token_hash, role{organizer,member,guest}, joined_at)`
- `availability(trip_id, member_id, date, state{yes,maybe,no})`
- `proposals(id, trip_id, kind{stay,activity,food,transport,other}, title, url, preview_json, est_cost, notes, status{open,locked,rejected}, vote_deadline, created_by)`
- `votes(proposal_id, member_id, value{up,down,neutral})`
- `itinerary_items(id, trip_id, proposal_id NULLABLE, day, start_time, title, location, url, notes, sort)`
- `expenses(id, trip_id, paid_by_member, amount_cents, currency, description, split_json /*[{member,share}]*/, at)`
- `comments(id, trip_id, target{proposal,itinerary_item,expense}, target_id, member_id, body, at)`
- `activity(id, trip_id, member_id, verb, target, payload_json, at)` · `nudges(id, trip_id, member_id, kind, due_at, sent_at)`

## 8. Feature specs

**F1 — Trip + magic membership.** Create trip → invite link `/j/<token>`; opening it asks only for a name (guest) with optional account upgrade later (guest → user merge keeps votes/expenses). *AC:* join-to-first-vote < 30 s on mobile; guest upgrading to account retains all their data (merge tested).

**F2 — Date matcher.** Per-member availability grid over a candidate range; heatmap of overlap; "best windows" ranked (most yes, fewest no, length-matching organizer's target nights). Organizer locks a window → trip status `locked`, calendar invites (ICS) emailed. *AC:* windows ranking correct on property-tested fixtures; ICS imports cleanly into Google/Apple Calendar.

**F3 — Proposals & deadline voting.** Paste URL → unfurled card (image/title/price if detectable); votes visible live; deadline auto-closes → status suggestion (lock if >50% up and organizer confirms). Non-voters get exactly 2 nudges (24 h and 2 h before deadline). *AC:* deadline close + nudges fire within 1 min of schedule (worker test with fake clock); vote tallies exclude departed members.

**F4 — Itinerary.** Day columns; drag-drop items; locked proposals appear in an "unplaced" tray; each item: time, place (free text + maps link), attached URL/notes. Print/share read-only view. *AC:* reorder persists and syncs live to a second client < 2 s; read-only share link works logged-out.

**F5 — Expenses.** Add expense, choose payer + split (equal/custom/subset); balances view + "simplify debts" (min-transaction settlement algorithm); mark settled. *AC:* settlement algorithm property test — settlements sum to balances, transaction count ≤ n−1; currency is per-trip single currency (enforced).

**F6 — Engagement loop.** Trip home shows "needs your action" (unvoted proposals, unfilled availability); weekly digest email of activity; day-before-event reminder with itinerary. *AC:* digests only send when there is activity; unsubscribes honored.

## 9. Milestones

- **M0 (1):** scaffold, auth (organizer accounts + guest tokens), trip CRUD + invite links.
- **M1 (2):** date matcher end-to-end incl. lock + ICS. *Ships: better-than-Doodle date picking.*
- **M2 (3):** proposals, unfurl, voting, deadlines + nudge worker. *Ships: decision funnel.*
- **M3 (4):** itinerary + comments + activity feed/SSE. *Ships: MVP.*
- **M4 (5):** expenses + settlement, digests, PWA polish + push, read-only shares. *Ships: v1.0.*

## 10. Testing

- Unit: window-ranking, settlement, guest-merge.
- Worker tests with injected clock (deadlines, nudges, digests).
- Playwright: full happy path — create → 3 guests join → availability → lock → propose → vote → itinerary → expense → balances; plus mobile-viewport run.
- Unfurl fetcher: SSRF guard tests (blocks private IPs/redirect tricks) — release-blocking.

## 11. Risks & open questions

- Group tools die when one person won't engage → guest-no-account flow + nudges are the mitigations; measure join→vote conversion from day 1.
- Link unfurling of Airbnb/booking sites may be bot-blocked → graceful fallback to manual title/price entry.
- Expense trust (no receipts/payments) is fine for friends-scale; explicitly out of scope to stay un-regulated.
