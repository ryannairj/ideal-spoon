# Implementation Spec — Molt (tower defense)

Read `PLAN.md` first. This file fixes the implementation decisions; do not re-decide them.

## 0. Fixed decisions

| Decision | Value |
|---|---|
| Name | **Molt** · License MIT (code) / CC-BY for own art |
| Engine | Phaser 4 + TS 5 + Vite; React 18 DOM overlay for menus/draft/panels |
| Sim | fixed 30 tps, decoupled from render; sim code in `src/sim/` **must not import Phaser** (enforced by eslint boundary rule) |
| PRNG | `seedrandom` (alea), all randomness through injected `Rng` |
| Damage types | `kinetic, fire, frost, shock, toxin` (+ `true`) |
| Resist cap | 70% soft (Normal caps 50%), trait floors per PLAN |
| Adaptation constants (initial tuning) | @TUNE(adapt_rate=0.10) (Normal 0.07, Hard 0.13), @TUNE(decay=0.03), molt every 3rd wave locks top resist as trait with @TUNE(traitFloor=0.30); all calibrated by balance harness (§6) |
| Grid | square cells 64 px; maps 20×12; path via baked flow field per map (generated at build from map JSON) |
| Save format | JSON in localStorage keys `molt.meta`, `molt.run` (run = seed + input log) |
| Replay/share code | `base64url(deflate(JSON{seed, mapId, inputLog}))` |
| Content | all in `content/*.json`, zod-validated in CI; effect DSL interpreted (no eval) |
| Towers (6 families) | Bolt(kinetic), Ember(fire), Glacier(frost), Arc(shock), Spore(toxin), Prism(true, expensive) — 3 tiers each |
| Enemies (12) | runner, brute, shelled(shock-pop shield), burrower, splitter, healer, flyer, swarmling, tank, leech, mirrorling, warden + bosses Broodmother, Mirror |
| Economy | bounty flat per species; interest 10% of banked gold per wave, cap 50; sell 70% |

## 1. Repository layout

```
molt/
  src/
    sim/                      # deterministic core (no Phaser)
      world.ts                # tick loop, entity arrays (SoA-lite)
      spawn.ts pathing.ts targeting.ts projectiles.ts
      damage.ts               # pipeline: base→armor→resist→riders→ledger
      evolution.ts            # R vector update, molts, traits
      economy.ts waves.ts     # budget-based generator
      cards.ts effects.ts     # effect DSL interpreter
      rng.ts types.ts save.ts replay.ts
    render/                   # Phaser scenes & view of sim state
      GameScene.ts HudScene.ts sprites.ts vfx.ts audio.ts
    ui/                       # React overlay
      {MainMenu, RunSetup, DraftModal, ThreatPanel, TowerPanel, RecapScreen,
       MetaScreen, SettingsModal, DailyCard}.tsx
    bridge/store.ts           # zustand mirror of sim snapshots for React
    main.ts
  content/{towers.json, enemies.json, waves/, cards.json, biomes.json, maps/*.json, DESIGN.md}
  tools/
    balance/{bots.ts, runner.ts, report.ts}   # headless sims (node, imports src/sim only)
    validate-content.ts  bake-flowfields.ts
  test/ (vitest)  e2e/ (playwright)
```

## 2. Dependencies

`phaser@4`, `react`, `react-dom`, `zustand`, `seedrandom`, `pako` (deflate), `zod`, `howler` (audio), `tailwindcss`. Dev: `vitest`, `playwright`, `eslint` + `eslint-plugin-boundaries` (sim isolation rule), `tsx` (balance runner).

## 3. Sim core contracts

```ts
// world.ts
interface SimInput { tick: number; op: 'place'|'upgrade'|'sell'|'target_mode'|'start_wave'|'pick_card'; …payload }
class World { static fromSeed(seed: string, mapId: string, content: Content): World
  applyInput(i: SimInput): void   // inputs only valid between waves or build-legal mid-wave
  tick(): void                    // 1/30 s
  snapshot(): Snapshot            // render/UI read-model (positions, hp, R vector, gold…)
  hash(): string                  // FNV over canonical state — replay determinism tests
}
```
Damage pipeline order (damage.ts): `raw = base × towerMods` → armor flat reduce (min 1) → `× (1 − R[type])` (skip for `true`) → riders (burn DoT, slow, chain −25%/hop max 4, toxin vuln +4%/stack max 10 — vuln applies *after* resist) → ledger `killAttribution[type] += isKillingBlow ? 1 : 0`.

Evolution (evolution.ts, end of wave): `share[t] = kills[t]/totalKills`; `R[t] = clamp(R[t] + adapt_rate×share[t] − decay×(1−share[t]), 0, cap)`; every 3rd wave: top R type → trait floor (telegraph stored one wave earlier in `pendingMolt` — UI reads it). Counter-cards can `strip_trait` or `decay×2 for 2 waves`.

## 4. Content schemas (zod, excerpt)

```ts
Tower = { id, family, tier: 1|2|3, cost, range, rateMs, damage: {type, base},
  projectile: 'instant'|'ballistic'|'beam', riders: Rider[], upgradesTo?: id }
Rider = {kind:'burn', dps, ms} | {kind:'slow', pct, ms} | {kind:'chain', hops}
      | {kind:'vuln', stacks} | {kind:'pierce', count}
Enemy = { id, hp, speed, armor, baseResists: Partial<R>, bounty, traits: EnemyTrait[], flying?: true }
Card  = { id, kind:'tower'|'mutation'|'relic', rarity:1|2|3, effect: Effect[] }  // Effect = typed DSL list
Wave  = generator params: { budget(n) = 40 + 22n + 3n², speciesWeights per biome, boss waves [15] }
```
CI checks: every EnemyTrait has ≥1 counter tower/card (`content/DESIGN.md` matrix parsed and asserted).

## 5. UI/Scene inventory

MainMenu → RunSetup (3 seeded map offers, loadout pick) → GameScene+HudScene with React ThreatPanel (R bars + trend arrows + pendingMolt banner + last-wave kill pie), TowerPanel (stats incl. per-type DPS after current R), DraftModal (1-of-3 between waves), RecapScreen (evolution area-chart from ledger history, kill stats, cores earned), MetaScreen (unlock tree: 6 families visible, 2 locked at start + 4 loadouts), DailyCard (date-seed run, local best), SettingsModal (speed default, colorblind palette toggle, screen-shake, volume). Touch: tap-place with confirm ✓/✗ chip, 44 px min targets.

## 6. Balance infrastructure (build in M2, not later)

`tools/balance/bots.ts`: strategies `monoFire`, `greedyDps`, `rotator3` (rotates top-3 uncounter-ed types), `banker`. `runner.ts --runs 200 --strategy all --difficulty normal` → `report.md` (win rate, median leak wave, gold curves). CI nightly job commits report artifact. Tuning PRs must include before/after report (checked by CI comment script).

## 7. Milestone task lists

**M0** — T1 Vite+Phaser+React scaffold + eslint sim-boundary; T2 world tick + one map (JSON + baked flow field) + runner enemy walking path; T3 Bolt tower placing/firing (instant), gold/lives, wave start button; T4 snapshot→render bridge; T5 determinism test (seed → 1000 ticks → hash) in CI.
**M1** — T1 damage pipeline + all 5 types + riders + ledger; T2 remaining 5 families ×3 tiers (content + upgrade UI); T3 8 base species + traits (shelled, splitter, healer, flyer, burrower); T4 wave generator (budget curve) + 15-wave run loseable/winnable; T5 targeting modes; T6 economy (interest, sell).
**M2** — T1 evolution.ts + tests (share math, cap, decay); T2 ThreatPanel + pendingMolt telegraph (always exactly 1 wave early — test); T3 molt traits + strip/decay counter-effects; T4 Prism true-damage family + toxin vuln interplay tests; T5 **balance harness** + first tuning pass (rotator3 must beat monoFire — CI gate `monoFire winrate < 20%, rotator3 > 60%` on Normal).
**M3** — T1 cards.json (40) + effects interpreter + DraftModal; T2 run structure (RunSetup seeded offers, save/resume via input-log replay, RecapScreen w/ evolution chart); T3 meta unlocks (cores per run: wave reached + boss bonus) + MetaScreen; T4 save-format versioning + migration stub.
**M4** — T1 bosses (Broodmother spawn-on-hit-type logic; Mirror reflect: top damage type dealt this wave → 20% reflected as tower stun); T2 remaining species (tank, leech, swarmling, mirrorling, warden); T3 maps 2–5 + biome params (evolution rates); T4 difficulty tiers wiring; T5 counter-matrix CI check green.
**M5** — T1 juice: hit tints per type, resist "ding" indicator (icon + sound when `resistApplied > 0.3`), molt beat (1.5 s slow-mo + shader tint), SFX set, shake toggle; T2 colorblind palette + type icons everywhere colors appear; T3 daily seed card + replay share codes (encode/decode + watch mode driving World from log); T4 itch.io/web deploy workflow; T5 5-person playtest quiz protocol doc + fixes. Tag v1.0.

## 8. Test mapping

CI per-PR: determinism hash, content validation, counter-matrix, unit suites (damage order, evolution math, economy, effects DSL), playwright smoke (place → survive wave 1). Nightly: balance report. Replay tick-perfection test: recorded 15-wave input log → final hash equality.

## 9. Visual/audio feedback spec (M5 T1 — clarity is a mechanic)

Because reading adaptations is core play (PLAN F6), feedback thresholds are fixed, not vibes:

| Cue | Trigger | Visual | Audio |
|---|---|---|---|
| Hit tint | every projectile hit | 80 ms flash on enemy sprite, colored by the tower's damage type (palette in `render/vfx.ts`) | per-type impact tick |
| **Resist "ding"** | `resistApplied ≥ @TUNE(resistDingThreshold=0.30)` on a hit (i.e. ≥ 30% of that hit's damage absorbed by R) | small shield-glyph puff over the enemy in the resisted type's color + damage number drawn struck-through/greyed | distinct dull "clink" (howler), rate-limited to 1/enemy/250 ms so swarms don't cacophony |
| Molt beat | wave where a trait locks in | 1.5 s slow-mo (sim unaffected — render-time-scale only) + carapace shader tint sweep across swarm | rising "molt" sting |
| Pending-molt telegraph | one wave before a molt (`pendingMolt` set) | ThreatPanel banner + pulsing arrow on the trending type bar | soft warning tone once on wave start |
| Colorblind mode | settings toggle | each damage type also carries a distinct icon shape shown everywhere color conveys meaning (bars, tints, ding glyph, TowerPanel) | — |

Acceptance (PLAN F6 AC): in the 5-person playtest quiz, a resisting hit is correctly identified ≥ 90% of the time; the quiz protocol doc (M5 T5) shows 10 recorded clips (5 resisted, 5 clean) and scores identification. `resistApplied` is emitted in the sim snapshot per hit event so the render layer never recomputes resist math.
