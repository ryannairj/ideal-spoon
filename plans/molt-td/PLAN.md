# Molt — a tower defense where the swarm evolves against you

**Category:** Game · **Difficulty:** Medium–Hard · **Platform:** Web (desktop + touch), Steam wrapper later

## 1. The new idea

Classic TD has a solved meta: find the best tower, spam it. **Molt's twist: enemies evolve resistances to whatever is killing them.** Every wave, the swarm "molts" — damage types that scored the most kills last wave get partially resisted next wave (visible as carapace colors). Overuse fire, and wave 8 arrives fireproof. The game is not about the best build; it's about **rotating pressure** across a damage-type economy, reading the swarm's adaptations, and timing your "purge" moments before resistances lock in. Wrapped in a roguelike run structure: draft towers/mutations between waves, biomes with different evolution rates, permadeath runs, meta-unlocks.

Secondary twist that reinforces the first: **you can see the swarm's gene pool** — a small "threat panel" shows current resistance distribution and what it's trending toward, so counterplay is informed, not random.

## 2. Why it can win

- One-sentence hook that screams "new": *"a TD where the creeps adapt to you."* Streams well; every run's meta is different by construction.
- Evolution pressure elegantly fixes TD's core staleness (dominant strategies) with a system, not with content volume.

## 3. Players & core loop

Roguelike/TD fans (Bloons, Rogue Tower, Brotato audience). Session = 1 run, 20–30 min, 15 waves + boss.

Loop: place/upgrade towers → wave → observe kill-attribution + evolution report → draft 1 of 3 cards (new tower / mutation / relic) → adapt → repeat. Lose = leak limit exceeded; meta-currency unlocks new tower families and starting loadouts.

## 4. MVP scope & non-goals

**MVP ("vertical slice"):** 1 biome, fixed map pool (5 maps), 6 tower families × 3 upgrade tiers, 5 damage types, evolution system, 12 enemy species + 2 bosses, draft system (~40 cards), run structure + meta-unlock screen, local saves, seeded runs, sound + juice pass.
**Non-goals:** multiplayer, level editor, mobile app stores (touch-friendly web only), procedural maps (fixed, hand-tuned), narrative.

## 5. Tech stack

- **Engine:** TypeScript + **Phaser 4** (2D, batteries included, easy web ship). ECS-lite architecture on top (own lightweight components) for sim entities.
- **Deterministic sim:** fixed-timestep (30 tps) simulation decoupled from render; all randomness through a seeded PRNG (`seedrandom`) — enables replays, daily seeds, and headless balance sims.
- **UI:** Phaser for game scene; React overlay (DOM) for menus/draft/threat panel.
- **Persistence:** localStorage + export/import save code. No backend in MVP (daily seed = date-derived seed client-side).
- **Art:** flat-shaded vector/atlas style (placeholder from kenney.nl-style CC0 packs, consistent palette), shader tint for resistance carapace colors.

## 6. Core systems design

**Damage types (5):** Kinetic, Fire, Frost, Shock, Toxin. Each has a signature secondary (pierce, burn DoT, slow, chain, stacking vuln).

**Evolution system (the heart):**
- Swarm has a resistance vector `R = {kinetic, fire, frost, shock, toxin}`, each 0–70% (hard cap).
- After each wave: kill-attribution table (damage type → % of kills) feeds adaptation: `R[t] += adapt_rate × kill_share[t] − decay × (1 − kill_share[t])`, clamped; `adapt_rate` scales by biome/difficulty; decay rewards rotating away.
- **Molt moments:** every 3rd wave the swarm "locks in" its top resistance as a *trait* (e.g. Fireproof Carapace: fire resist floor 30%) — telegraphed one wave ahead in the threat panel, giving a purge window.
- Counter-tools: some towers deal `true` (unresistable, expensive) damage; some cards *strip* traits; Toxin's vuln-stacking partially ignores resists — escape valves so runs can't go unwinnable, tuned so they're supplements not mains.

**Draft cards:** towers, tower mutations (change a tower's damage type or add a rider), relics (global rules: "resistance decay ×2", "molts happen every 4th wave"). Rarity-weighted, seeded.

**Economy:** gold per kill (flat per enemy, no damage-type bias), interest on savings (Kingdom Rush-style banking tension), sell = 70%.

## 7. Data & content pipeline

All content as data files (`content/*.json`, schema-validated in CI): towers, enemies, waves, cards, biomes. Balance lives in data, not code.

- `towers.json`: id, family, tier, cost, range, rate, damage{type, base}, projectile, riders[], upgrade paths.
- `enemies.json`: id, hp, speed, armor, base_resists, bounty, traits[], flying/ground.
- `waves.json` per map: composition curves + budget-based generator params (wave N budget → spend on species mix, seeded).
- `cards.json`: kind, rarity, effect DSL (small typed effect list, interpreted — no eval).

## 8. Feature specs

**F1 — Sim core.** Deterministic fixed-step: spawning, pathing (per-map baked flow field), targeting (first/strong/close configurable per tower), projectiles, damage pipeline (base → armor → resist → riders → attribution ledger). *AC:* same seed + same input log ⇒ bit-identical end state (replay test in CI); 500 enemies + 60 towers ≥ 55 fps render on a 2020 laptop (sim ≤ 8 ms/tick headless).

**F2 — Evolution + threat panel.** As designed above; panel shows resist bars, trend arrows, next-molt forecast, last wave's kill attribution. *AC:* headless balance sim proves: mono-type strategies fail by wave ~10 at default tuning while 3-type rotation clears (automated strategy bots, see §10); trait telegraphs always appear exactly one wave early.

**F3 — Build & upgrade UX.** Grid placement with range preview + path-block validation, upgrade/sell panel, DPS-by-type stat readout, fast-forward ×2/×3, pause-and-place. *AC:* touch targets ≥ 44 px; misplacement undo within 3 s refunds 100%.

**F4 — Draft & run structure.** 1-of-3 draft between waves; run screen (map select from 3 seeded options, loadout); death/victory recap showing evolution history graph + kill stats; meta-unlocks (spend cores on new families/loadouts). *AC:* recap graph matches the attribution ledger; unlocks persist across sessions; save-and-quit mid-run restores exactly (uses replay determinism).

**F5 — Bosses & species variety.** Boss 1 "Broodmother" (spawns adapted minions on hit-type), Boss 2 "Mirror" (reflects your top damage type at towers). Species traits: shielded (immune until shield popped by Shock), burrower (skips 20% of path), splitter, healer, flying. *AC:* each trait has a designed counter present in the MVP tower set (matrix documented in `content/DESIGN.md` and covered by bot sims).

**F6 — Juice & clarity pass.** Hit flashes tinted by damage type, resist "ding" indicator when hitting a resisting enemy (crucial feedback), molt cutscene beat (1.5 s), SFX set, screen-shake (toggleable), colorblind-safe type palette + icons. *AC:* a resisting hit is visually distinguishable in a 5-person playtest quiz ≥ 90% of the time.

**F7 — Daily seed + replay share.** Date-seeded daily run; end screen gives a share code (seed + input log, compressed) anyone can watch as a replay. *AC:* replay of a shared code reproduces the run tick-perfect.

## 9. Milestones

- **M0 (1):** Phaser + React scaffold, fixed-step sim harness, one map, one tower, one enemy, gold/lives loop. 
- **M1 (2):** damage pipeline + 5 types, attribution ledger, 6 tower families, 8 species, wave generator. *Ships: competent classic TD.*
- **M2 (3):** evolution system + threat panel + molts/traits + counter-tools; headless bot-sim balance harness. *Ships: the hook, playable.*
- **M3 (4):** draft cards, run/meta structure, saves, recap. *Ships: roguelike loop complete.*
- **M4 (5):** bosses, remaining species, 5 maps, difficulty tiers.
- **M5 (6):** juice/SFX/colorblind pass, daily seed, replays, itch.io/web release. *Ships: v1.0 vertical slice.*

## 10. Testing & balance infrastructure

- **Headless balance sims are first-class:** scripted strategy bots (mono-fire, greedy-DPS, 3-type rotator, banker) run 200 seeded runs each in CI-nightly; output win-rate/leak curves per wave. Tuning changes must include before/after sim report.
- Determinism/replay test on every PR (seed → hash of end state).
- Content JSON schema validation + "every enemy trait has ≥ 1 counter" static check.
- Playwright smoke: load → place tower → survive wave 1.

## 11. Risks & open questions

- Balance is the real work; the bot-sim harness (M2) exists precisely so tuning is empirical, not vibes. Budget 30% of total effort post-M2 for tuning.
- Evolution could feel punishing to new players → difficulty tiers scale `adapt_rate`, and "Normal" caps resists at 50%.
- Phaser perf with 500+ entities → object pooling + single spritesheet from day 1; if it still binds, sim already headless → render layer swappable to Pixi without sim rewrite.
