# Repwell — local-first workout tracking with progressive-overload intelligence

**Category:** Life improvement · **Difficulty:** Medium · **Platform:** Mobile-first PWA, local-first with optional sync

## 1. Vision

A lifting-first workout tracker that gets out of the way *in the gym* (huge touch targets, zero-latency logging, works in a basement with no signal) and earns its keep *between* sessions: it watches your history and tells you exactly what to attempt next ("last week 3×8@60 kg felt easy → try 62.5 kg"), flags stalls, and auto-suggests deloads. Strong/Hevy territory, but local-first, ad-free, and opinionated about progression.

## 2. Why it can win

- Most trackers are databases; the coaching layer (double-progression logic, stall detection, deload timing) is what beginners+intermediates actually need and is deterministic — no AI required, so it's fast, free, and explainable ("why this suggestion" always shows the rule).
- Local-first PWA: instant everywhere, data is yours (SQLite in the browser via OPFS), sync optional.

## 3. Users & use cases

Gym-goers following structured programs (SL5x5, PPL, 531-style, custom).

1. Start "Push Day" → each exercise pre-filled with target sets/reps/weight from the progression engine → log a set in one tap (target was right) or two (adjust).
2. Rest timer auto-starts per exercise config; notifies at 0.
3. Review: e-1RM trends, volume per muscle group per week, PR history.
4. Build a program from templates or from scratch; deload week auto-inserted after stall.

## 4. MVP scope & non-goals

**MVP:** exercise library (seeded ~250 exercises + custom), program builder, workout logging, rest timer, progression engine (double progression + linear + percentage-based), stall/deload logic, analytics (e-1RM, volume, PRs), full offline, export (CSV/JSON), optional encrypted sync.
**Non-goals:** cardio/GPS, nutrition (see meal-tracker plan), social/feeds, wearables, AI form checks, Apple/Google Health integration (v1.1).

## 5. Tech stack

- **App:** React + TypeScript + Vite PWA; SQLite-WASM on OPFS via `sqlocal`/`wa-sqlite`; service worker for full offline.
- **Sync (optional):** tiny Go server storing E2E-encrypted database snapshots + oplog (libsodium sealed boxes, key from passphrase); last-write-wins per row with per-device oplog merge (single-user, low-conflict domain).
- **Charts:** lightweight (uPlot). **Notifs:** Web Push + vibration for rest timer.

## 6. Architecture

All logic client-side. Domain layer (pure TS, fully unit-tested): progression engine, stall detector, analytics computations — isolated from UI and storage. Storage layer: repository over SQLite. Sync layer: background push/pull of oplog when enabled.

## 7. Data model (client SQLite)

- `exercises(id, name, muscle_groups[], equipment, kind{barbell,dumbbell,machine,bodyweight,cable}, bar_weight, increment, is_custom)`
- `programs(id, name, schedule_json /*day templates*/)` · `program_exercises(program_id, day, exercise_id, scheme_json /*sets×reps or %-based*/, progression{double,linear,percent,manual}, rest_sec, sort)`
- `workouts(id, program_id NULLABLE, day_label, started_at, finished_at, notes, feeling{1..5})`
- `sets(id, workout_id, exercise_id, set_no, weight, reps, rpe NULLABLE, is_warmup, is_pr, logged_at)`
- `progression_state(exercise_id, program_id, current_target_json, consecutive_fails, last_deload_at)`
- `oplog(seq, table, row_id, op, payload, device_id, at)` for sync.

## 8. Feature specs

**F1 — Logging UX (make or break).** Workout screen: exercise cards with target rows pre-filled; tap row = logged as-target (haptic); long-press = adjust steppers (weight by exercise increment, reps ±1); plate calculator on weight tap; previous-session ghost values shown. *AC:* logging a prescribed 5×5 session = ≤ 7 taps total; every interaction < 50 ms perceived (no network, optimistic writes); works in airplane mode end-to-end.

**F2 — Progression engine.** Rules as pure functions: **double progression** (hit top of rep range all sets → +increment, reset to bottom range), **linear** (+increment each session, fail → repeat, 3 fails → deload 10%), **percent-based** (targets from training max; AMRAP feeds TM adjustments). Every suggestion carries a `because` string. *AC:* golden-table unit tests per rule (≥ 40 cases incl. edge: missed session, manual override, unit switch); suggestions match tables exactly.

**F3 — Stall & deload.** Stall = rule-specific fail-streak or e-1RM flat/declining over N sessions → banner suggests deload week (auto-generated: −40% volume) or scheme change; user accepts/dismisses (tracked). *AC:* detector fires on fixture histories exactly per spec; never fires during the first 2 weeks of a program.

**F4 — Rest timer.** Per-exercise duration, auto-start on set log, persistent notification with ±30 s, vibrate/sound at zero, survives screen lock (Web Push fallback where background timers throttle). *AC:* fires within ±2 s with screen locked on Android Chrome and iOS PWA (document iOS caveats honestly if platform limits).

**F5 — Analytics.** Per exercise: e-1RM (Epley) trend, best set, PR timeline; per week: sets per muscle group vs. evidence-based target bands (10–20), streak calendar. *AC:* computations property-tested; charts render 2 years of data < 200 ms.

**F6 — Programs & library.** Template gallery (5 canonical programs seeded), drag-drop builder, exercise substitutions (same muscle group suggestions). *AC:* a seeded template starts logging correctly with zero configuration.

**F7 — Data freedom.** CSV/JSON export, Strong/Hevy CSV import, encrypted sync opt-in (passphrase, multi-device). *AC:* export→wipe→import round-trip is lossless; Strong sample files import correctly (fixtures); sync server never sees plaintext (test: server-side grep for known exercise name fails).

## 9. Milestones

- **M0 (1):** PWA scaffold, SQLite-OPFS storage, exercise library, free-form logging. 
- **M1 (2):** programs + targeted logging UX + rest timer. *Ships: gym-usable.*
- **M2 (3):** progression engine + stall/deload + "because" UI. *Ships: the differentiator.*
- **M3 (4):** analytics + PRs + import/export.
- **M4 (5):** encrypted sync server + multi-device, template gallery polish, iOS PWA hardening. *Ships: v1.0.*

## 10. Testing

- Domain layer: exhaustive unit + property tests (progression monotonicity, unit-conversion round-trips kg/lb).
- Playwright mobile-viewport: full workout offline, then sync reconciliation between two browser contexts.
- Storage: OPFS persistence across reload/eviction scenarios; migration tests from v(n−1) schemas.

## 11. Risks & open questions

- iOS PWA storage eviction → prompt "install + enable sync/export" early; document; sync is the real safety net.
- SQLite-WASM maturity quirks → wrap storage behind repository interface; IndexedDB fallback path if OPFS fails.
- Rep-range religion varies → progression rules are data-configurable per program exercise; defaults follow the templates.
