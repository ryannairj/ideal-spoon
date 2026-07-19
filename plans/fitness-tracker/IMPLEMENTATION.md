# Implementation Spec — Repwell (fitness tracker)

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Repwell** · License MIT (local-first, community trust) |
| App | React 18 + TS 5 + Vite PWA; NO Next.js (pure client + tiny sync server) |
| Storage | SQLite WASM via `wa-sqlite` on OPFS; repository layer `src/data/repo.ts` is the only DB touchpoint; IndexedDB fallback driver if OPFS unavailable |
| Domain layer | pure TS in `src/domain/` — zero imports from UI/data (eslint boundary) |
| Sync server | Go 1.23 single binary `repwell-sync`, SQLite, port 8720; stores E2E-encrypted oplog batches (libsodium sealed box, key from passphrase argon2id) |
| Sync model | per-device oplog (row-level LWW by `updated_at`, device tiebreak); client merges; server is dumb encrypted mailbox |
| Units | store kg + reps canonically; lb display via setting; increments per exercise (default barbell 2.5 kg / dumbbell 2.0 / machine 5.0) |
| e1RM | Epley: `w × (1 + reps/30)`, reps ≤ 12 only (above → excluded from e1RM trend) |
| Progression rules (fixed ids) | `double`, `linear`, `percent`, `manual` — pure functions in `domain/progression/` |
| Deload | trigger: rule-specific fail-streak (3) OR e1RM slope ≤ 0 over 6 sessions (min 14 days in program); deload template −40% volume for 1 week |
| Seeded content | `content/exercises.json` (~250), `content/programs/*.json` (SL5x5, PPL, 531-ish "5/3/1 base", GZCLP, full-body-3day) |
| Charts | uPlot · Rest timer | Notification API + vibration; iOS limits documented in-app |
| Ports | dev 5173, sync 8720 |

## 1. Repository layout

```
repwell/
  app/
    src/
      domain/
        progression/{double.ts, linear.ts, percent.ts, index.ts, types.ts}
        stall.ts analytics.ts (e1RM, volume/muscle-week, PRs) units.ts platecalc.ts
      data/{driver.ts (wa-sqlite/OPFS), schema.sql, migrations/, repo.ts, oplog.ts, sync.ts, exportImport.ts}
      screens/{Home.tsx, Workout.tsx, ExercisePicker.tsx, ProgramList.tsx, ProgramBuilder.tsx,
               ExerciseDetail.tsx (charts/PRs), Analytics.tsx, Settings.tsx, ImportExport.tsx}
      components/{SetRow.tsx (tap=log, long-press=steppers), RestTimerBar.tsx, PlateCalcSheet.tsx,
                  GhostValues.tsx, SuggestionCard.tsx ("because" popover), DeloadBanner.tsx,
                  StreakCalendar.tsx, MuscleVolumeBars.tsx}
      sw.ts  main.tsx  router.tsx (react-router)
    content/{exercises.json, programs/*.json}
  sync/  (Go server: main.go, store.go, auth.go — ~500 lines)
  fixtures/{strong-export.csv, hevy-export.csv, histories/*.json}
```

## 2. Dependencies

App: `react, react-router-dom, wa-sqlite, zustand, uplot, zod, idb-keyval (settings cache), libsodium-wrappers, fractional-indexing, date-fns, tailwindcss, vite-plugin-pwa`. Dev: `vitest, fast-check, playwright`. Sync: stdlib + `chi`, `modernc.org/sqlite`.

## 3. Client schema (`schema.sql`)

```sql
CREATE TABLE exercises (id TEXT PK, name TEXT, muscle_groups TEXT, equipment TEXT,
  kind TEXT CHECK(kind IN ('barbell','dumbbell','machine','bodyweight','cable')),
  bar_weight REAL DEFAULT 20, increment REAL, is_custom INTEGER DEFAULT 0,
  updated_at INTEGER, deleted INTEGER DEFAULT 0);
CREATE TABLE programs (id TEXT PK, name TEXT, days TEXT /*json [{label, sort}]*/,
  updated_at INTEGER, deleted INTEGER DEFAULT 0);
CREATE TABLE program_exercises (id TEXT PK, program_id TEXT, day_label TEXT, exercise_id TEXT,
  scheme TEXT /*json: {sets, repsMin, repsMax} | {percentTable}*/, progression TEXT,
  rest_sec INTEGER DEFAULT 120, sort TEXT, updated_at INTEGER, deleted INTEGER DEFAULT 0);
CREATE TABLE workouts (id TEXT PK, program_id TEXT, day_label TEXT, started_at INTEGER,
  finished_at INTEGER, notes TEXT, feeling INTEGER, updated_at INTEGER, deleted INTEGER DEFAULT 0);
CREATE TABLE sets (id TEXT PK, workout_id TEXT, exercise_id TEXT, set_no INTEGER,
  weight_kg REAL, reps INTEGER, rpe REAL, is_warmup INTEGER DEFAULT 0, is_pr INTEGER DEFAULT 0,
  logged_at INTEGER, updated_at INTEGER, deleted INTEGER DEFAULT 0);
CREATE TABLE progression_state (program_id TEXT, exercise_id TEXT, target TEXT /*json*/,
  consecutive_fails INTEGER DEFAULT 0, last_deload_at INTEGER, training_max_kg REAL,
  updated_at INTEGER, PRIMARY KEY(program_id, exercise_id));
CREATE TABLE oplog (seq INTEGER PK AUTOINCREMENT, tbl TEXT, row_id TEXT, op TEXT,
  payload TEXT, device_id TEXT, at INTEGER, synced INTEGER DEFAULT 0);
CREATE TABLE meta (key TEXT PK, value TEXT);
```

## 4. Progression contracts (`domain/progression/types.ts`)

```ts
interface ProgressionInput { scheme; state: ProgState; lastSessions: SessionResult[]; increment: number }
interface Suggestion { sets: {weightKg, targetReps}[]; because: string }
double:  all sets hit repsMax last session → weight += increment, target repsMin; else same weight,
         target = min(lastBestReps+…progress within range); fail(below repsMin any set) ×3 → deload −10% & reset fails.
linear:  +increment per completed session; any failed set → repeat; 3 consecutive fails → −10% deload.
percent: targets = trainingMax × table[week]; AMRAP last set: reps ≥ table.amrapTarget+5 → TM +increment×2,
         ≥ target → TM +increment, < target−2 → TM −increment.
```
Golden tables in `fixtures/histories/` (≥ 40 cases incl. missed session ⇒ no double-increment, manual override sets state, kg/lb switch leaves canonical kg untouched).

## 5. Sync protocol

`POST /v1/register {email?, passphraseProof}` → account id; `POST /v1/push {deviceId, batch: sealedBox(oplog rows)}`; `GET /v1/pull?since=serverSeq` → batches from other devices. Client merge: apply row if `updated_at` newer (tie → deviceId lexical); conflicting concurrent edits to same set-row are last-writer, acceptable domain-wise. Server never decrypts (test greps server DB for exercise names).

## 6. Key UX contracts

Workout screen: exercise cards in day order; SetRow tap = log exactly target (haptic + 50 ms optimistic write), long-press = stepper sheet (weight by increment, reps ±1, RPE optional); ghost text = last session same exercise/set_no; auto-start RestTimerBar on log (per-exercise rest_sec, persistent notification w/ ±30 s, vibrate at 0). PlateCalcSheet on weight tap (bar + available plates setting, per-side output). Finish → summary (volume, PRs flagged via analytics.ts, feeling picker). 5×5 prescribed session = ≤ 7 taps E2E test.

## 7. Milestone task lists

**M0** — T1 Vite PWA scaffold + wa-sqlite driver + migrations + repo layer + eslint boundaries; T2 exercises content + picker + free-form logging (ad-hoc workout); T3 offline verification (airplane E2E) + CI.
**M1** — T1 programs content + builder (day tabs, add exercise, scheme editor, fractional sort); T2 targeted Workout screen (cards, SetRow, ghosts) + ≤7-taps test; T3 rest timer (+lock-screen behavior doc + platform tests); T4 plate calc.
**M2** — T1 progression functions + golden suites + fast-check (monotonicity, unit round-trip); T2 SuggestionCard + "because" popover wired to next-session targets; T3 stall.ts (fail-streak + e1RM slope; not-in-first-2-weeks rule) + DeloadBanner (accept → generated deload week; dismiss tracked); T4 progression_state persistence + manual override UI.
**M3** — T1 analytics.ts (e1RM trends, weekly muscle volume vs 10–20 band, PR timeline, streak calendar) + uPlot screens (2 y data < 200 ms perf test); T2 CSV/JSON export + Strong/Hevy importers (fixture-tested) + lossless round-trip test.
**M4** — T1 sync server (Go) + client oplog push/pull + merge + two-context Playwright reconciliation test; T2 passphrase setup UX + key derivation (libsodium argon2id) + server-plaintext grep test; T3 iOS PWA hardening pass (storage persist API request, install nag when >50 workouts unsynced); T4 template gallery polish + onboarding (pick program → first workout in <60 s). Tag v1.0.

## 8. Test mapping

Domain: golden + property suites (release gate). Storage: OPFS persistence across reload, migration from v(n−1), IndexedDB fallback smoke. Playwright mobile: full offline workout; sync divergence. Timer: ±2 s assertion with page-hidden emulation where driver allows; manual device checklist committed for iOS.
