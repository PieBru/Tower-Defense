# Tower Defense Game — System Prompt (v3)

Create a **single, self-contained HTML file** that implements a polished tower defense game. Everything — HTML, CSS, and JavaScript — must live in one file with zero external dependencies. The game should open and run immediately in any modern browser.

> **v2 changelog (high level):** finishes the set-bonus and elemental-cascade systems that v1 left as display-only; fills in four adjacent combos that had no defined effect; and adds six new systems aimed squarely at replayability — Relics (roguelike per-run picks), Tower Fusion, Kill-Streak/Crit juice, Special Enemy behaviors + Boss Phases, a support tower (Alchemist), Endless Mode, and a Daily Challenge + streak system. Everything below is the full spec — not a diff — so it can be used standalone.

---

## Architecture Principles
- **Single file, no build step.** All code in one `.html` file. No frameworks, no libraries, no CDN links.
- **Canvas-based rendering** at 800x600 for the game map. Use `requestAnimationFrame` with delta-time physics (never assume 60fps).
- **Web Audio API** for all sound effects — synthesized tones via `OscillatorNode`, never load external files. Always call `ensureAudio()` on first user gesture to satisfy autoplay policies.
- **`localStorage` for persistence** — save best scores, prestige points, prestige upgrades, daily-streak data, and endless-mode high score across sessions.
- **Delta-time movement** — all speeds multiplied by `(dt / 16)` so the game runs identically at 30fps or 144fps.
- **Game speed multiplier** — a `speedMult` value (1x/2x/3x, player-toggleable) multiplies the `dt` passed into `update()` only. `draw()` always renders at real time. This keeps physics deterministic while making the game feel faster.
- **No `setInterval` for game logic** — use `requestAnimationFrame` only. Use `setInterval` only for UI polling (e.g., wave-complete detection).

---

## Core Gameplay Loop
1. Player places towers on open areas of the map by clicking.
2. Player clicks "Next Wave" to start enemy waves (an enemy-composition preview is visible before committing).
3. Enemies follow a path from left to right. Towers auto-target and shoot. Kills build a **kill-streak meter**; shots have a small chance to **crit**. Rare **Golden Enemies** may spawn mid-wave.
4. Defeated enemies drop money. Money is earned from kills, streak bonuses, and perfect-wave bonuses.
5. Between waves, player chooses one perk from 4 random options. Every 5th wave, player *also* chooses one permanent-for-this-run **Relic** from 3 options.
6. Every 3 maxed (Lv.3) towers of the same type can be **fused** into a single Ultimate tower.
7. Game over when lives reach zero. Player can restart, open the prestige shop, or (if the campaign was cleared) continue into **Endless Mode**.

---

## Map & Path System
- **3 distinct map layouts** selectable via a dropdown before/during gameplay:
  - **Winding Path** — classic zigzag with multiple direction changes
  - **Serpent** — long, multi-lane winding path covering most of the canvas
  - **Crossroads** — two paths that share the first half then diverge
- Paths are defined as **arrays of waypoints**. Compute segment lengths at build time for distance-based positioning.
- Use `posAt(dist, segmentIndex)` to get (x, y) for any point along the path — `segmentIndex` is optional (defaults to 0) and used for multi-path maps like Crossroads. Enemies and projectiles are positioned via `posAt(enemy.dist, enemy.si)`.
- Draw paths with: base color + subtle border glow + dashed center line + start/end emoji markers (🏁 -> 🏠).
- **Placement validation:** reject clicks too close to the path (distance < ~20px from any path point) or too close to existing towers.
- **Daily Challenge** reuses these same three maps but locks map + wave RNG to a seeded PRNG derived from the calendar date (see *Daily Challenge & Streaks*).

---

## Tower System

### Tower Types (6 total)
Each tower has: name, cost, range, base damage, fire rate, color, projectile color/speed, element type, and an ability.

| Type | Cost | Element | Role |
|------|------|---------|------|
| **Basic** 🔵 | 25 | Arcane | Balanced all-rounder |
| **Sniper** 🔮 | 50 | Fire | Long range, high single-target damage |
| **Rapid** 🟡 | 35 | Lightning | Short range, very fast fire rate |
| **Ice** ❄️ | 30 | Ice | Slows enemies in range on hit |
| **Bombardier** 💥 | 45 | Fire | AoE splash damage projectiles |
| **Alchemist** 🧪 | 40 | *None (support)* | No direct damage — buffs nearby towers |

#### Alchemist (NEW — support tower)
- Deals **no damage** and does not benefit from or count toward elemental cascades (it has no element).
- Passively grants all towers within its range a **+15% damage, +10% fire rate** aura (multiplicative, stacks with synergy/set/cascade bonuses — see damage formula below).
- Ability **[Y] Catalyst**: instantly refreshes 50% of the cooldown on all abilities of towers in range. Cooldown 25s.
- Still gets 3 upgrade levels (aura strength scales the same way as damage: +35% per level, applied to the aura percentages instead of raw damage).
- Alchemist has its own Set Bonus (3+ towers: aura range +25%) but is excluded from Adjacent Combos and Elemental Cascades since those are element-driven.

### Tower Upgrades (3 levels per tower)
- Click a placed tower to open an upgrade panel showing current stats, synergy bonus, set progress, and ability info.
- Each level increases: **+35% damage, +20% range, -18% fire rate** (exponential decay).
- Upgrade costs scale: `base_cost * (0.6 + level * 0.5)`.
- Sell refunds ~50% of total invested (base cost + upgrade payments).
- Level shown as colored dots above the tower (gray -> gold -> red).
- A Lv.3 tower shows a small **fusion-ready icon** once 2 other Lv.3 towers of the same type exist on the map.

### Tower Abilities (Active Skills) + Level Upgrades
Each tower has a unique active ability triggered by keyboard shortcut or clicking the ability button in the ability bar. **Abilities scale with tower level** — each upgrade level improves the ability's power, range, or secondary effects.

| Tower | Key | Ability | Effect | Cooldown |
|-------|-----|---------|--------|----------|
| Basic | Q | **Arcane Blast** | Fires multiple projectiles at nearest enemies simultaneously | 8s |
| Sniper | E | **Armor Pierce** | Next shot deals scaled damage, ignores elemental resistance | 12s |
| Rapid | R | **Chain Lightning** | Lightning chain hits N enemies for X% damage each | 10s |
| Ice | F | **Blizzard** | Freezes all enemies in an area for Y seconds (zero movement) | 15s |
| Bombardier | T | **Firestorm** | Massive AoE fire damage on a target area | 20s |
| Alchemist | Y | **Catalyst** | Refreshes 50% cooldown on all abilities of towers in range | 25s |

- Ability cooldown shown as a **red arc ring** around the tower + countdown number.
- Ready state: **golden pulsing ring** + golden border on ability button.
- Prestige upgrade "Quick Cast" reduces all cooldowns by 15% per level (max 3 levels = 45%).

#### Ability Upgrades Per Tower Level
Abilities improve as the tower is upgraded, unlocking new behaviors or scaling existing ones:

| Tower | Lv.1 | Lv.2 Upgrade | Lv.3 Upgrade |
|-------|------|-------------|-------------|
| **Basic** [Q] Arcane Blast | 3 projectiles, 1.5x base dmg | 5 projectiles, 2.0x base dmg | 7 projectiles, 2.5x base dmg + chain to 1 extra enemy |
| **Sniper** [E] Armor Pierce | 3x damage, ignores resistance | 4x damage, ignores resistance | 5x damage, ignores resistance + piercing (hits through target to next) |
| **Rapid** [R] Chain Lightning | 5 chains, 60% dmg per chain | 7 chains, 70% dmg per chain | 9 chains, 80% dmg per chain + each chain splashes 1 nearby enemy for 30% |
| **Ice** [F] Blizzard | Freeze 3s in radius 80 | Freeze 4s in radius 100 + slow 50% outside freeze zone | Freeze 5s in radius 120 + enemies frozen >2s take 10 dmg/s (shatter) |
| **Bombardier** [T] Firestorm | 60 dmg in radius 70 | 80 dmg in radius 85 | 100 dmg in radius 100 + leaves burning ground (15 dps for 4s) |
| **Alchemist** [Y] Catalyst | Refresh 50% cd, range 90 | Refresh 65% cd, range 110 | Refresh 80% cd, range 130 + grants next ability used by each buffed tower a free (no-cooldown) recast |

- Level upgrades are visual: tower gains an extra ring around the ability cooldown indicator.
- Lv.3 abilities show a distinct color flash when activated (gold for Basic, crimson for Sniper, violet for Alchemist, etc.).
- Ability upgrade info shown in the upgrade panel: "Lv.2: 5 projectiles" or "Lv.3: +chain to 1 extra".

### Tower Fusion / Evolution (NEW)
Rewards committed single-type builds with an endgame sink instead of leaving maxed towers idle.
- When **3 Lv.3 towers of the same type** exist on the map, a **⚗️ Fuse** button appears on each of their upgrade panels.
- Fusing consumes all 3 towers (they're removed from the map) and places one **Ultimate tower** of that type at the position of the tower the player clicked to initiate fusion. No refund — this is a deliberate, permanent transformation, not a sale.
- Ultimate towers have: 2.2x the stats of a Lv.3 tower, a passive unique to each type (e.g., Ultimate Bombardier's splash never friendly-fires other towers' targeting priority; Ultimate Ice permanently slows — never fully recovers — enemies that exit its range for 2s), and their ability has **no cooldown limit removed but gains +1 charge** (can be queued twice before going on cooldown).
- Ultimate towers **still count toward Set Bonuses** as 1 tower of their base type (so fusing can drop you below a set threshold — a real strategic tradeoff worth surfacing in the confirmation prompt: "Fusing will reduce your Basic Set count to 2 (below the 3-tower threshold). Continue?").
- Ultimate towers **cannot be upgraded further** and sell for only 25% of invested value (steep loss discourages fuse-then-sell abuse).
- Visual: Ultimate towers are ~30% larger with a rotating outer ring in the element's color.

### Synergy Bonuses
- Placing **same-type towers adjacent** (within ~70px) triggers stacking bonuses: **+20% damage, +10% range per neighbor** (capped at 3 neighbors = +60%/+30%).
- Prestige upgrade "Allied Spirit" boosts base synergy to +30%/+15%.
- Visual: **golden pulsing glow** around towers with active synergy.

### Tower Set Bonuses (3+ Same Type)
Placing **3 or more towers of the same type** anywhere on the map activates a set bonus. This rewards committed builds and encourages clustering strategies. **All six sets now have real gameplay effects** (v1 only implemented Basic; Sniper/Rapid/Ice/Bombardier/Alchemist were display-only placeholders):

| Set | Count Bonus | Effect |
|-----|------------|--------|
| 🔵 **Basic Set** (Arcane) | 3 towers | All Basic towers gain +30% range |
| 🔵 **Basic Set** | 5 towers | Arcane Blast ability fires +2 projectiles per use |
| 🔮 **Sniper Set** | 3 towers | +15% chance for Sniper shots to crit (see Kill-Streak & Crit System) |
| 🔮 **Sniper Set** | 5 towers | Crits from Sniper towers apply Armor Pierce's resistance-ignore for free |
| 🟡 **Rapid Set** | 3 towers | Fire rate increases by 2% per consecutive hit on the same target, capped at +20%, resets when target changes |
| 🟡 **Rapid Set** | 5 towers | Chain Lightning gains +2 chain targets |
| ❄️ **Ice Set** | 3 towers | Slowed enemies take +15% damage from all towers |
| ❄️ **Ice Set** | 5 towers | Blizzard ability cooldown -30% |
| 💥 **Bombardier Set** | 3 towers | Splash radius +20% |
| 💥 **Bombardier Set** | 5 towers | Firestorm leaves burning ground even at Lv.1/Lv.2 (normally a Lv.3-only effect) |
| 🧪 **Alchemist Set** | 3 towers | Aura range +25% |
| 🧪 **Alchemist Set** | 5 towers | Aura also grants +10% ability damage/effect strength |

- Set bonuses are cumulative: 3-tower bonus active at 3+, 5-tower bonus is additional.
- Visual: **set glow** — colored aura around each set tower, pulsing.
- Set progress shown in upgrade panel: "Set Bonus: 🔵 Basic (3/5)" with progress bar.
- Sets can be mixed strategically — e.g., 3 Basic + 3 Sniper = two active sets, or 5 Rapid = maximum power.
- Set bonuses apply globally to all towers of that type on the map, not just clustered ones. A player can build a sacrificial 3rd tower elsewhere to activate a set bonus without sacrificing positioning.

---

## Elemental System

### Elements
| Element | Color | Icon |
|---------|-------|------|
| Arcane | `#74b9ff` | 🔵 |
| Fire | `#e17055` | 🔥 |
| Lightning | `#fdcb6e` | ⚡ |
| Ice | `#81ecec` | ❄️ |
| *(Support — Alchemist)* | `#a29bfe` | 🧪 *(no elemental interactions)* |

### Enemy Affinities (appear from wave 3+)
| Enemy | Resistance | Weakness | Notes |
|-------|-----------|----------|-------|
| Flameguard 🛡️ | Fire (40% dmg) | — | Appears wave 3+ |
| Frostshell 🧊 | Ice (40% dmg) | — | Appears wave 4+, tanky |
| Stormhide ⛈️ | Lightning (40% dmg) | — | Appears wave 5+ |
| Voidborn 🌀 | Arcane (40% dmg) | — | Appears wave 6+, very tanky |
| Boss 👹 | None | **Fire (2x dmg)** | Every 3rd wave, final enemy, now has **phases** (see Special Enemies) |
- Damage multiplier: **super effective = 2.0x**, **resisted = 0.4x**, neutral = 1.0x.
- "Armor Pierce" ability bypasses resistance entirely.
- "Elemental Mastery" perk (available as wave perk) boosts super-effective damage to 2.5x.
- Show 🛡️ icon near resistant enemies so players can identify counters.

### Elemental Reaction Cascades (3+ Elements) — now with real effects
When towers of **three different elements** are placed within a cluster (all three within ~100px of each other), they trigger an elemental reaction cascade. **v1 shipped these as visual-only; v2 gives each a concrete combat effect.**

| Reaction | Elements Required | Icon | Effect |
|----------|------------------|------|--------|
| **Steam Vent** | Fire + Ice | 💨🔥❄️ | Enemies hit by either element within the cluster's radius take +20% damage from the *other* element for 3s |
| **Overcharge** | Lightning + Fire | ⚡🔥 | Burning enemies hit by Lightning attacks explode for 15 bonus damage to nearby enemies |
| **Glacial Storm** | Ice + Lightning | ❄️⚡ | Slowed enemies have a 10% chance per Lightning hit to be instantly frozen for 1s |
| **Arcane Surge (Triple)** | Arcane + any two others | 🔵🔥❄️ | All towers in the cluster gain +10% fire rate |
| **Elemental Convergence** | All 4 elements | 🌈 | All towers in the cluster gain +15% damage AND +10% fire rate (does not stack additively with Arcane Surge — Convergence supersedes it while all 4 are present) |

- Cascades require **all element towers alive and in range**. If one is sold, cascade deactivates.
- Cascades stack with each other (except the Arcane Surge / Convergence overlap noted above) — multiple distinct cascades can be active simultaneously from different clusters.
- Visual: **rainbow shimmer** around active reaction clusters; a small floating icon of the reaction's name appears briefly when a cascade first activates.
- Cascade damage/rate bonuses are applied in the central damage formula (see Technical Implementation Notes) so they interact correctly with synergy/set/Alchemist-aura bonuses rather than being calculated separately.

---

## Special Enemies & Boss Phases (NEW)

v1 enemies differed only by stat multipliers (HP/speed/reward). v2 adds behavioral variety so *how* an enemy dies matters, not just its numbers.

| Enemy | Appears | Behavior |
|-------|---------|----------|
| **Shielded** 🔷 | Wave 4+ | Has a separate shield HP pool (30% of max HP) that must be depleted before the health bar drops; shield regenerates if the enemy goes 4s without being hit |
| **Splitter** 🟢 | Wave 6+ | On death, spawns 2 smaller copies at 25% HP and 120% speed each (copies do not split again) |
| **Healer** ✚ | Wave 7+ | Periodically (every 3s) heals the lowest-HP enemy within 60px for 8% of that enemy's max HP; should be visually distinct so players learn to prioritize it |
| **Teleporter** 🌀 | Wave 8+ | Every 6s, has a chance to blink forward 40-80px along the path, partially bypassing tower range windows |
| **Golden Enemy** ✨ | Any wave, rare (~4% spawn chance per wave, capped at 1 per wave) | Fast, fragile (low HP), drops a lump bonus (80-150 money) or a bonus Relic Shard if killed within 4s of spawning; escapes permanently (no life lost, just a missed reward) if not killed in time |

### Boss Phases
Bosses (every 3rd wave) are no longer a flat HP bar:
- **Phase 1 (100%→50% HP):** normal behavior.
- **Phase transition at 50% HP:** boss becomes briefly shielded (immune) for 1.5s, screen shake + distinct sound, then either **spawns 2-3 add enemies** (scaled to current wave) or **gains a temporary damage-resistance shield** (absorbs a fixed amount of damage before breaking), chosen based on wave number (adds on early bosses, shield on later ones, to teach both patterns).
- **Phase 2 (50%→0% HP):** boss speed increases 15%.
- This turns "every 3rd wave" into a readable event with a beat, rather than just a bigger sponge.

---

## Kill-Streak & Critical Hit System (NEW)

Adds moment-to-moment variable-reward juice on top of the strategic layer.

- **Kill Streak Meter:** each kill increments a streak counter and grants a small escalating money bonus (`+1 money per streak stage, capped`). The streak decays if **no kill occurs within 2.5s** (linear decay, doesn't hard-reset instantly, so near-misses don't feel punishing). Losing a life also resets the streak.
- Streak milestones (10/25/50/100) trigger a brief on-screen banner, a distinct ascending sound, and a temporary global +5% fire rate buff for 6s ("Overdrive") — stacks up to 2 concurrent Overdrive buffs.
- **Crit chance:** base 5% on all towers, modified by Sniper Set bonus, relics, and prestige upgrades. Crits deal 1.75x damage, show a larger/bolder floating damage number in a distinct crit color, and trigger a small screen flash + higher-pitched hit sound.
- Prestige upgrade **"Momentum"** (new — see Prestige Shop) increases streak decay time and base crit chance permanently.

---

## Relic System (NEW — roguelike per-run progression)

The perk system (per-wave, forgotten after each wave) is good for short-term tactical choices but doesn't build a run identity. Relics sit alongside it as the long-term hook.

- **When offered:** every 5th wave (waves 5, 10, 15...), immediately after the normal perk modal is resolved, a second modal offers **3 random Relics** — pick 1.
- **Persistence:** Relics last for the **entire run** (unlike perks, which reset each wave) and stack with each other.
- Relics are drawn from a pool larger than what any single run will see, so different runs feel different. Example pool (aim for ~16-20 total in implementation):

| Relic | Icon | Effect |
|-------|------|--------|
| Ember Core | 🔥 | Fire-element towers deal +20% damage |
| Frozen Heart | ❄️ | Ice-element towers slow 15% more effectively |
| Overcharged Coils | ⚡ | Lightning-element towers gain +15% fire rate |
| Arcane Battery | 🔵 | Arcane-element towers' abilities cooldown 20% faster |
| Bloodmoney | 💰 | +1 money per enemy kill, flat, regardless of enemy type |
| Steady Hands | 🎯 | +10% base crit chance |
| Twin Strike | 🗡️ | 10% chance for any shot to fire twice (second shot at 50% damage) |
| Extra Plot | 📐 | Reduces minimum tower-placement spacing by 15%, allowing tighter clusters |
| Salvage Expert | 🔧 | Sell refund increased from 50% to 65% |
| Adrenaline | 💉 | Kill-streak decay is 40% slower |
| Convergent Will | 🌈 | Elemental cascades trigger at 2 elements instead of 3 (does not affect Convergence's 4-element requirement) |
| Fused Potential | ⚗️ | Fusion requires only 2 maxed towers of a type instead of 3 |
| Lucky Find | 🍀 | Golden Enemy spawn chance doubled |
| Iron Lives | ❤️ | +5 max lives, applied immediately |
| Alchemist's Gift | 🧪 | Alchemist aura range and strength both +25% |
| Second Wind | 🌀 | Once per run, automatically triggers when lives would hit 0: refills to 3 lives instead of game over (consumed on use) |

- Relic panel is always visible in a corner of the HUD (small icon row with tooltips) so the player can see their build taking shape.
- Relics reset at the start of a new run (they are **not** permanent progression — that's what the Prestige Shop is for). This distinction should be clear in the UI copy ("Relics — this run only" vs "Prestige — permanent").

---

## Endless Mode (NEW)

- Unlocked once the player has cleared wave 20 in a normal run (or immediately available from the main menu after the first campaign clear, whichever is simpler to implement).
- After the last defined wave, the game continues generating waves using the same scaling formulas extrapolated indefinitely (no cap on wave number).
- Tracked with its own **separate best score** in `localStorage` (`td_endless_best`), shown on the HUD as `♾️ Endless Best: Wave X` distinct from the normal best score.
- Relics and perks continue to be offered on the same cadence. Prestige points continue to accrue from wave milestones.
- This exists purely as a "floor is lava, how far can I go" loop for players who've mastered the core game — no new mechanics required, just removing the ceiling.

---

## Combo Chain System

### Adjacent Combos (2 Towers)
When two towers of specific types are placed **adjacent** (~80px), they activate a combo. Combos persist as long as the towers remain adjacent. Active combos are shown in the combo panel. **v1 left four of these five combos without a defined effect ("—"); v2 fills them all in:**

| Combo | Towers | Effect | Visual |
|-------|--------|--------|--------|
| 🧊⚡ **Freeze Zone** | Ice + Rapid | Rapid's fast hits apply a stacking slow charge; the 8th stack on a target forces a 1s hard freeze, then resets | Cyan sparks between the two towers |
| 💨🔥 **Steam Burn** | Ice + Bombardier | AoE attacks apply fire DoT (8 dps for 3s) on hit | Orange flicker on affected enemies |
| 🔵🔮 **Arcane Surge** | Basic + Sniper | Each Basic hit on a target reduces that Sniper's ability cooldown by 0.3s (marks the target); Sniper crits on marked targets deal +25% | Blue tracer line connecting a Basic hit to the nearest Sniper |
| ⚡💥 **Lightning Rod** | Rapid + Bombardier | Bombardier splash leaves a "charge" on hit enemies; the next Rapid hit on a charged enemy detonates it for 20 bonus damage to nearby enemies | Small yellow spark icon over charged enemies |
| ❄️🔵 **Ice Fortress** | Ice + Basic | Basic towers deal +25% damage to slowed or frozen enemies | Basic tower gains a faint icy tint while combo is active |

- **Combo panel** (top-left of canvas) shows all possible combos, set progress, and elemental cascades organized in sections:
    - ⚡ Adjacent Combos (checkmarks for active)
    - 🏗️ Set Progress (e.g., "🔵 Basic 3/5")
    - 🌈 Elemental Cascades (active reactions with element icons and their effect on hover/tap)
- Towers with active combos get a **purple glow ring**. Towers with active sets get a colored aura. Towers in cascades get rainbow shimmer.
- Steam Burn DoT shown as `🔥X/s` text above affected enemies.
- Frozen enemies get a cyan ice shell visual and cannot move.

---

## Enemy System

### Enemy Properties
Each enemy has: distance along path, current HP, max HP, base speed, reward, size, color, optional elemental affinity, and an optional `behavior` tag (`shielded` / `splitter` / `healer` / `teleporter` / `golden` / none).

### Wave Generation
- Count scales: `3 + wave` enemies per wave (4 on wave 1, 5 on wave 2, etc.).
- HP scales: `25 + wave * 12`.
- Speed scales: `1.2 + min(wave * 0.06, max_speed)`.
- Reward scales: `4 + floor(wave * 1.2)`.
- Spawn interval: `max(600, 1500 - wave * 90)` ms (slower early, faster late).
- Intra-wave spacing: `max(36, ((40 - wave * 2) - i * 8) * 3)` units cumulative distance. Head = wide spread, tail = tight cluster.
- **Boss every 3 waves** — final enemy, 4x HP, 0.6x speed, 3x reward, has phases (see Special Enemies & Boss Phases).
- **Speedsters every 2nd wave** — 50% HP, 1.9x speed, smaller size.
- **Armored from wave 5** — 2x HP, 0.8x speed, appears periodically.
- **Shielded from wave 4+, Splitter from wave 6+, Healer from wave 7+, Teleporter from wave 8+** — each appears periodically once unlocked, roughly 1-2 per wave, scaled up slightly on later waves.
- **Golden Enemy** — independent roll each wave, ~4% chance, capped at 1 per wave (see above).
- Elemental affinities assigned based on wave number and position in queue.
- **Next-wave preview:** before the player clicks "Next Wave," a small row of icons (reusing enemy emoji/colors) shows the upcoming wave's composition — including any boss, special-behavior enemies, and whether a Relic pick follows this wave — so the player can plan tower placement and ability timing in advance.

### Status Effects on Enemies
| Effect | Trigger | Visual | Behavior |
|--------|---------|--------|----------|
| Slow | Ice tower hit, Blizzard ability, Glacial Storm cascade | Cyan aura | Speed multiplied by slowFactor (0.45 base) |
| Freeze | Blizzard ability, Freeze Zone combo, Glacial Storm cascade | Cyan ice shell + white border | Zero movement until duration expires |
| Burn DoT | Steam Burn combo on hit, Firestorm Lv.3 burning ground, Overcharge cascade explosions | Orange flicker | Damage per second applied continuously |
| Shield (enemy-side) | Innate to Shielded enemy type | Blue hexagon overlay | Absorbs damage into a separate shield pool before HP is affected; regenerates after 4s unhit |
- Slow effect decays over time (linear interpolation back to 1.0).
- Frozen enemies completely stop — they don't advance along the path.
- Status effect durations shown in ability descriptions.

---

## Perk System (Between Waves)

After each wave completes, show a modal with **4 random perks** chosen from:

| Perk | Icon | Effect |
|------|------|--------|
| Double Cash | 💰💰 | 2x money from kills this wave |
| Iron Will | ❤️❤️ | +3 lives this wave |
| Rush Order | ⚡ | Free next wave (no money cost) |
| Reinforce | 🔧 | All towers boost +1 level this wave |
| Scavenger | 🔍 | Find +50 money on ground |
| Frost Zone | ❄️ | Ice towers have 2x range this wave |
| Elemental Mastery | 🔮 | All towers deal +50% to weak enemies |
| Ability Surge | ⚡ | Abilities cost no cooldown this wave |
| Lucky Strike | 🎲 | +15% crit chance this wave |
| Golden Rush | ✨ | Guarantees a Golden Enemy will spawn this wave |

- Perks are **per-wave** (reset after the wave ends) — this is intentionally different from Relics, which persist for the whole run.
- Modal shows 4 cards in a 2x2 grid. Click one to select it.
- Reinforce applies before the wave starts. Scavenger adds money immediately.
- Every 5th wave, the Perk modal is followed by the Relic modal (see Relic System).

### Perfect Wave Bonus (NEW)
- If a wave completes with **zero lives lost during that wave**, award a small bonus: `+15 money` and `+1 Relic Shard` fragment (4 shards = 1 free bonus Relic pick, tracked in the HUD as a small shard counter). This rewards precision play over brute-force turret spam without requiring a whole new currency.

---

## Prestige System (Permanent Progression)

Earn **5 prestige points per prestige level** (each level = every 5 waves completed). On game over: `pp += (Math.floor(wave/5) - currentPrestigeLevel) * 5`. Points persist across sessions via `localStorage`.

Note: The original spec said "1 point per 5 waves" — the actual implementation awards 5 points per level. This was changed during development to give the player more flexible spending power in the expanded 9-upgrade shop.

### Prestige Shop (between runs)
Spend points on permanent upgrades:

| Upgrade | Icon | Cost | Effect | Max Levels |
|---------|------|------|--------|------------|
| Rich Start | 💰 | 2 per level | +30 starting money | 3 |
| Power Boost | ⚔️ | 2 per level | +15% all damage | 4 |
| Fortify | 🛡️ | 3 per level | +5 starting lives | 3 |
| Bulk Buy | 🏷️ | 2 per level | -10% tower costs | 3 |
| Swift Souls | 💨 | 3 per level | Enemies -25% slow effect | 2 |
| Allied Spirit | 🤝 | 3 per level | +10% synergy bonus | 2 |
| Quick Cast | ⚡ | 3 per level | Abilities -15% cooldown | 3 |
| **Momentum** *(NEW)* | 🔥 | 3 per level | +5% base crit chance, streak decay -20% | 3 |
| **Relic Seeker** *(NEW)* | 🔮 | 4 per level | Start each run with 1 free random Relic per level | 2 |

- Costs scale: `base_cost * (current_level + 1)`.
- Max level shows "MAX" with green text.
- Prestige level shown in HUD as `✨ Prestige Lv.X`.
- Points displayed as `💎 X/5` (5 points = new prestige level).

---

## Daily Challenge & Streaks (NEW)

Gives players a reason to open the game every day without requiring any server/multiplayer infrastructure.

- **Daily seed:** derive a numeric seed from the current date string (e.g., `20260702`) and feed it into a small seeded PRNG (e.g., mulberry32) that replaces `Math.random()` for map selection, wave enemy composition, golden-enemy timing, and perk/relic offers **only while Daily Challenge mode is active**. This guarantees every player sees the identical run on a given day.
- Daily Challenge is a distinct mode selectable from the main menu; it uses the player's current prestige upgrades (progression carries over) but tracks a **separate score** in `localStorage` per calendar day (`td_daily_YYYYMMDD`), so the player can compare "today's run" against their own best attempt at today's seed (re-attempts allowed, only the best is kept).
- **Daily streak counter:** on completing at least one Daily Challenge run, store `td_streak_count` and `td_streak_lastDate`. If the player returns on the very next calendar day, increment the streak; if a day is skipped, reset to 1. Streak grants a small escalating bonus to starting prestige points on that day's run (e.g., `+1 pp per 3 streak days, capped at +5`), shown in the HUD as `🔥 Streak: X days`.
- No server validation — this is a trust-the-client, `localStorage`-only feature, consistent with the rest of the game's persistence model. It's a retention nudge, not an anti-cheat system.

---

## Visual Polish

### Rendering Pipeline
```
requestAnimationFrame(timestamp)
  -> compute dt = min(timestamp - lastTime, 33)   // cap at ~30fps minimum
  -> update(dt * speedMult, now)                   // physics, AI, spawning — speedMult from 1x/2x/3x toggle
  -> draw()                                         // render everything, always real-time
```

### Visual Effects
- **Screen shake** — on kills, boss deaths, and boss phase transitions, shift canvas transform by random offset for 4-8 frames. Intensity scales with event severity.
- **Particle system** — spawn particles on hits (4 small), kills (12 medium), upgrades (15 large), crits (18, brighter), fusion (25, large burst), golden enemy kill (20, sparkly gold).
- **Floating damage numbers** — spawn at hit position, float upward with slight horizontal drift, fade out over 45 frames. Green for money gained, red for normal damage, bright yellow/larger font for crit damage.
- **Radial gradient enemies** — white highlight at top-left, color in middle, dark at bottom-right for a 3D sphere look. Golden Enemy uses a shimmering gold gradient with a subtle pulse.
- **Tower glow rings** — subtle colored aura around each tower. Golden pulse for synergy. Purple glow for active combos. Colored aura for sets. Rainbow shimmer for cascades. Soft violet aura ring for towers currently buffed by an Alchemist.
- **Projectile trails** — white inner dot + colored outer circle for depth.
- **Background** — radial gradient from lighter center to darker edges, plus subtle 40px grid lines at low opacity.
- **Path rendering** — wide base stroke + semi-transparent border glow + dashed center line.
- **Kill-streak banner** — brief center-top banner + number pop animation on streak milestones (10/25/50/100).
- **Boss phase transition** — boss briefly flashes white/shielded, camera shake, radial shockwave ring drawn from the boss position.
- **Relic pick modal** — 3-card layout (vs perks' 2x2), each card with a distinct icon glow, styled slightly differently from the perk modal (e.g., violet border) so players immediately register "this one's permanent."

### Color Palette
| Purpose | Color | Hex |
|---------|-------|-----|
| Background | Dark slate | `#2d3436` -> `#363a4a` |
| Panel bg | Deep navy | `#16213e` |
| Accent green | Success | `#00b894` |
| Accent red | Danger | `#d63031` |
| Accent gold | Warning/Points | `#fdcb6e` |
| Accent blue | Info | `#74b9ff` |
| Accent violet | Relics / Alchemist | `#a29bfe` |
| Accent bright yellow | Crit / Golden Enemy | `#ffeaa7` |
| Text primary | Light | `#eee` |
| Text secondary | Muted | `#b2bec3` |

---

## UI Layout
```
+-------------------------------------------------------------+
|  HUD: money lives wave score best  |  streak  | shards | speed[1x 2x 3x] |
|  Prestige bar (if prestigeLevel > 0)   Daily streak (if in Daily mode)   |
+-------------------------------------------------------------+
|                                                               |
|                     GAME CANVAS                              |
|   +--- combo panel ---+                +-- upgrade panel --+  |
|   | Combo/Sets/Cascade |                | Tower Lv.3        |  |
|   | v Freeze Zone      |                | DMG: 45 (+20%)    |  |
|   |   Steam Burn       |                | Range: 120         |  |
|   +--------------------+                | Ability: Q...      |  |
|                                          | [Upgrade][Sell][⚗️Fuse]| |
|   +-- relic tray (small icon row, top-right under HUD) --+   |
|                                                               |
|              ability bar (when tower selected)                |
|   [Q icon 8s] Tower Name                                       |
|                                                               |
|   next-wave preview row (above Next Wave button, pre-wave)    |
+-------------------------------------------------------------+
|  Tower type buttons (incl. Alchemist) | Next Wave | Map select | Restart |
|  Status info text                                              |
+-------------------------------------------------------------+
```
- **HUD** — top bar with all core stats, plus the new kill-streak counter, perfect-wave shard counter, and the 1x/2x/3x speed toggle.
- **Combo panel** — top-left of canvas, organized in sections: Adjacent Combos, Set Progress, Elemental Cascades.
- **Relic tray** — small persistent icon row (top-right, under the HUD) showing active Relics for the run with hover/tap tooltips.
- **Upgrade panel** — top-right of canvas, appears when clicking a tower. Shows stats, synergy info, set bonus progress, ability details, and a Fuse button when eligible.
- **Ability bar** — bottom-center of canvas, appears when a tower is selected. Shows ability button with keybind, cooldown overlay, and name.
- **Next-wave preview row** — appears above the "Next Wave" button once the previous wave/perk/relic flow is resolved, showing icons for the upcoming wave's enemy composition.
- **Tower type buttons** — in the controls row, each showing icon + cost, now including Alchemist. Click to select placement mode.
- **Status text** — below canvas, shows contextual help ("Selected: Ice Lv.2 - press [F] for Blizzard").

---

## Sound Design
All sounds via Web Audio API oscillators. No files, no downloads.

| Event | Sound | Parameters |
|-------|-------|-----------|
| Tower shoot | Short square wave chirp | 880Hz, 60ms, vol 0.04 |
| Hit | Sawtooth thud | 320Hz, 80ms, vol 0.05 |
| Kill | Two-tone ascending | 600->900Hz square, 100+120ms |
| Wave start | Three-note arpeggio | 440->660->880Hz triangle, 150+150+200ms |
| Place tower | Sine pop | 600Hz, 80ms, vol 0.06 |
| Upgrade | Two-tone rise | 500->750Hz sine, 100+120ms |
| Sell | Triangle fade | 400Hz, 100ms, vol 0.05 |
| Error | Low sawtooth buzz | 200Hz, 150ms, vol 0.06 |
| Ice ability | High shimmer | 1200Hz sine, 150ms, vol 0.03 |
| AoE ability | Low boom | 150Hz sawtooth, 200ms, vol 0.06 |
| Ability activate | Three-tone fanfare | 600->900->1200Hz square, 80+60+120ms |
| Freeze | Crystal chime | 2000+1600Hz sine, 300+250ms, vol 0.04 |
| Combo activate | Ascending triangle | 500->750->1000Hz triangle, 80+80+160ms |
| **Crit hit** *(NEW)* | Bright double-chirp | 1000->1400Hz square, 40+40ms, vol 0.05 |
| **Streak milestone** *(NEW)* | Four-note rising run | 500->700->900->1100Hz triangle, 60ms each |
| **Golden enemy spawn** *(NEW)* | Sparkly high tremolo | 1800Hz sine w/ fast vol LFO, 400ms, vol 0.03 |
| **Golden enemy kill** *(NEW)* | Coin cascade | 900->1200->1500Hz square, 60ms each, vol 0.05 |
| **Relic pick** *(NEW)* | Deep resonant chime | 300+600Hz sine (dyad), 350ms, vol 0.05 |
| **Fusion** *(NEW)* | Rising sweep + boom | 400->1200Hz sawtooth 300ms, then 120Hz sawtooth 200ms, vol 0.07 |
| **Boss phase transition** *(NEW)* | Ominous low pulse | 100Hz square, 3x 150ms pulses, vol 0.06 |
| **Perfect wave** *(NEW)* | Clean major triad | 523+659+784Hz sine simultaneous, 300ms, vol 0.04 |

---

## Interaction Flow

### Placing a Tower
1. Click a tower type button (e.g., "Basic $25").
2. Cursor changes to crosshair. Status text confirms selection.
3. Click on map to place. Validates distance from path and other towers.
4. If invalid: play error sound, show rejection message.
5. If valid: deduct money, spawn tower, play place sound, update HUD.

### Selecting & Upgrading a Tower
1. Click on an existing tower (when not in placement mode).
2. Upgrade panel appears with stats, synergy info, set progress, ability details, and a Fuse button if 3 Lv.3 same-type towers exist.
3. Click "Upgrade" to spend money and increase level (up to 3).
4. Click "Sell" to refund ~50% of invested gold (25% for Ultimate/fused towers).
5. Click "Fuse" (when eligible) to open a confirmation dialog warning about any set-bonus threshold it will drop below, then consume the 3 towers and spawn an Ultimate tower.
6. Press the tower's ability key (Q/E/R/F/T/Y) or click the ability button to trigger ability.
7. Click "Close" or click empty map space to deselect.

### Starting a Wave
1. Before clicking "Next Wave," the next-wave preview row shows upcoming enemy composition.
2. Click "Next Wave" button.
3. Enemies begin spawning at intervals along the path.
4. Towers auto-target nearest enemy in range and fire; kill streak and crit rolls apply per hit.
5. Wave completes when all enemies are defeated or escaped. If zero lives were lost this wave, the Perfect Wave bonus triggers.
6. Perk selection modal appears — choose 1 of 4 random perks.
7. On every 5th wave, the Relic modal appears next — choose 1 of 3 random Relics.
8. Perks/Relics apply and next wave can be started.

### Game Over
1. Lives reach zero — game over overlay appears (unless the "Second Wind" Relic auto-saves the run once).
2. Shows final wave, score, kill-streak peak, and whether it's a new best score (or new Endless/Daily best, if applicable).
3. If waves completed >= 5: show prestige points earned + "Open Prestige Shop" button.
4. If waves completed >= 20 (or Endless Mode was already unlocked): show "Continue in Endless Mode" as an option instead of ending the run outright.
5. Otherwise: just "Play Again" button.

### Restart
1. Resets all run state (money, lives, towers, enemies, wave, kill streak, active Relics, shard count).
2. Preserves prestige data (points, upgrades, best score, endless best, daily streak).
3. Map selection resets to default (unless Daily Challenge mode locks it to the seeded map).

---

## Technical Implementation Notes

### Distance-Based Positioning
```javascript
// Segments stored per path in pathSegs array
// pathSegs[i].s = [{f:[x,y], t:[x,y], l:len}, ...]
// pathSegs[i].tl = total length of that path
function posAt(d, si) {
  const ps = pathSegs[si || 0];
  if (!ps) return {x: 750, y: 300}; // fallback
  for (const sg of ps.s) {
    if (d <= sg.l) {
      const t = d / sg.l;
      return {x: sg.f[0] + (sg.t[0] - sg.f[0]) * t, y: sg.f[1] + (sg.t[1] - sg.f[1]) * t};
    }
    d -= sg.l;
  }
  return {x: ps.s[ps.s.length-1].t[0], y: ps.s[ps.s.length-1].t[1]}; // clamp to end
}
```
Enemies store `dist` (distance traveled along path) and `si` (segment index for multi-path maps). Each frame: `dist += speed * (dt/16)`. Position retrieved via `posAt(enemy.dist, enemy.si)`. This makes path-following trivial - no need for steering behavior.

### Tower Targeting
For each tower, iterate all alive enemies, find the one with minimum distance to tower position that is within range. Use squared distance (`dx*dx + dy*dy`) for comparison to avoid `Math.sqrt` in hot loop. Only compute actual distance when needed (for projectile homing). Alchemist towers skip targeting entirely — they only run the aura-application pass.

### Projectile Homing
Projectiles store a reference to their target enemy object. Each frame, get the enemy's current position via `posAt(enemy.dist, enemy.si)` and move toward it. If target dies, mark projectile for removal. This gives smooth homing without path prediction.

### Central Damage Formula (extended for v2)
All bonus multipliers now flow through one function so cascades, sets, Alchemist auras, Relics, and prestige upgrades compose predictably instead of being scattered across ability-specific code:
```javascript
function finalDamage(tower, enemy, baseDamage, isCrit) {
  let d = baseDamage;
  d *= getElementalMultiplier(tower.element, enemy.affinity);   // 2.0 / 0.4 / 1.0
  d *= (1 + tower.synergyBonus);                                 // adjacency
  d *= (1 + tower.setBonus);                                     // set thresholds
  d *= (1 + tower.cascadeBonus);                                 // active cascades touching this tower
  d *= (1 + tower.alchemistAuraBonus);                           // nearby Alchemist(s)
  d *= (1 + relicDamageBonus(tower, enemy));                     // active Relics
  d *= globalDmgBonus;                                           // prestige Power Boost etc.
  if (enemy.status.slowed || enemy.status.frozen) d *= (1 + iceSetSlowedDamageBonus); // Ice Set 3+
  if (isCrit) d *= critMultiplier;                                // 1.75x base
  return Math.round(d);
}
```
Recompute `synergyBonus`/`setBonus`/`cascadeBonus`/`alchemistAuraBonus` per tower once per frame (or on any placement/sell/fusion event) rather than caching indefinitely — see gotcha #7 from v1, which still applies and now also covers Alchemist auras.

### Combo/Combo Detection (O(n²))
Brute-force check of all tower pairs on placement and periodically:
```javascript
function getActiveCombos() {
  const active = [];
  for (let i = 0; i < towers.length; i++) {
    for (let j = i+1; j < towers.length; j++) {
      if (distance(towers[i], towers[j]) < COMBO_RANGE) {
        const combo = checkCombo(towers[i], towers[j]);
        if (combo && !active.find(a => a.name === combo.name)) active.push(combo);
      }
    }
  }
  return active;
}
```
With < 50 towers this is negligible. No optimization needed. Alchemist towers are excluded from `checkCombo` (no element).

### Set Bonus Detection (O(n²) by type)
Group towers by type, then count per group. Ultimate (fused) towers count as 1 of their base type:
```javascript
function getSetBonuses() {
  const groups = {};
  for (const tw of towers) {
    const name = TOWER_TYPES[tw.typeIdx].name; // fused/Ultimate towers keep their base typeIdx
    groups[name] = (groups[name] || 0) + 1;
  }
  return groups; // e.g., { Basic: 5, Sniper: 3, Ice: 2, Alchemist: 3 }
}
```

### Element-Damage Calculation
```javascript
function getElementalMultiplier(towerElement, enemyAffinity) {
  if (!towerElement || !enemyAffinity || !enemyAffinity.name) return 1; // Alchemist has no element -> always 1
  if (enemyAffinity.weakness && enemyAffinity.weakness.name === towerElement.name) return 2.0;
  if (enemyAffinity.resist && enemyAffinity.resist.name === towerElement.name) return 0.4;
  return 1;
}
```

### Cascade Detection (Connected-Component Clusters)
Cascades are detected by finding connected components of towers within 100px of each other (flood-fill style, O(n²)). All unique element types in the component are collected, then checked against cascade rules. Alchemist towers are excluded from cluster membership (no element) but do not block clustering between other towers. This correctly handles Elemental Convergence (requires all 4 elements) and allows multiple cascades to stack from the same cluster.
```javascript
function activeCascades() {
  const ac2 = []; let seen = new Set();
  for (let i = 0; i < towers.length; i++) {
    if (seen.has(i) || !towers[i].element) continue; // skip Alchemist
    const cluster = [i]; seen.add(i);
    for (let a = 0; a < cluster.length; a++) {
      for (let b = 0; b < towers.length; b++) {
        if (seen.has(b) || !towers[b].element) continue;
        if (distance(towers[b], towers[cluster[a]]) <= 100)
          { cluster.push(b); seen.add(b); }
      }
    }
    if (cluster.length < 3) continue;
    const els = [...new Set(cluster.map(ci => TOWER_TYPES[towers[ci].typeIdx].el))];
    if (els.length < 3) continue;
    const cas = checkCascades(els); // Convergence (4-el) MUST be checked before Arcane Surge (3-el) — see gotcha
    for (const c of cas)
      if (!ac2.find(a => a.name === c.name)) ac2.push(c);
  }
  return ac2;
}
```

### Fusion Detection & Application
```javascript
function fusionEligible(typeIdx) {
  const maxed = towers.filter(t => t.typeIdx === typeIdx && t.lv >= 3 && !t.isUltimate);
  return maxed.length >= 3 ? maxed.slice(0, 3) : null;
}
function fuseTowers(threeMaxedTowers, placementTarget) {
  // remove the 3 towers, recompute all set/synergy/cascade/alchemist bonuses,
  // spawn one isUltimate:true tower at placementTarget.pos with 2.2x Lv.3 stats
  // and the type's unique Ultimate passive.
}
```
Fusion must trigger a full recompute of synergy/set/cascade bonuses immediately afterward — removing 3 towers can drop a set below its threshold for every other tower of that type on the map, not just the fused ones.

### Kill-Streak & Crit
```javascript
// on kill:
streak++; lastKillTime = now; money += Math.min(streak, STREAK_CAP);
if ([10,25,50,100].includes(streak)) triggerStreakMilestone();
// per frame:
if (now - lastKillTime > 2500) streak = Math.max(0, streak - dt * 0.01); // gradual decay, not a hard reset
// on hit resolution:
const isCrit = Math.random() < totalCritChance; // base 0.05 + Sniper Set + Relics + prestige Momentum
```
Losing a life resets `streak` to 0 immediately (distinct from the natural decay-on-no-kills path).

### Golden Enemy Spawn
```javascript
// once per wave, before spawning begins:
if (Math.random() < goldenEnemyChance /* base 0.04, doubled by Lucky Find relic, forced by Golden Rush perk */) {
  scheduleGoldenEnemy(randomTimeWithinWave());
}
```
Golden Enemy despawns (escapes, no life lost) if not killed within 4s of appearing — this is a deliberate small window to keep it feeling special rather than trivially farmable.

### Boss Phase Trigger
```javascript
// checked each frame while a boss enemy is alive and not already in phase2:
if (!boss.phase2 && boss.hp <= boss.maxHp * 0.5) {
  boss.phase2 = true;
  boss.invulnUntil = now + 1500; // 1.5s shield window
  if (wave <= 9) spawnAdds(2 + Math.floor(wave/6)); else grantDamageShield(boss, wave * 15);
  boss.speed *= 1.15;
  triggerScreenShake(6); playSound('bossPhase');
}
```

### Daily Seed PRNG
```javascript
function mulberry32(seed) {
  return function() {
    seed |= 0; seed = seed + 0x6D2B79F5 | 0;
    let t = Math.imul(seed ^ seed >>> 15, 1 | seed);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  };
}
// Daily mode: const rng = mulberry32(Number(dateString.replace(/-/g,'')));
// replace all Math.random() calls relevant to run generation with rng() while in Daily mode.
```
Every system that uses `Math.random()` for wave/enemy/perk/relic/golden-enemy generation must accept an injectable RNG function so Daily mode can swap it in without duplicating logic.

### Persistence Pattern (extended)
```javascript
// Load on init
const prestigeUpgrades = JSON.parse(localStorage.getItem('td_pu') || '{}');
const bestScore = parseInt(localStorage.getItem('td_best') || '0');
const endlessBest = parseInt(localStorage.getItem('td_endless_best') || '0');
const streakCount = parseInt(localStorage.getItem('td_streak_count') || '0');
const streakLastDate = localStorage.getItem('td_streak_lastDate') || '';
// Save after changes
localStorage.setItem('td_pp', prestigePoints);
localStorage.setItem('td_pl', prestigeLevel);
localStorage.setItem('td_pu', JSON.stringify(prestigeUpgrades));
localStorage.setItem('td_best', bestScore);
localStorage.setItem('td_endless_best', endlessBest);
localStorage.setItem('td_streak_count', streakCount);
localStorage.setItem('td_streak_lastDate', todayDateString);
localStorage.setItem(`td_daily_${todayDateString}`, dailyRunScore);
```

### Ability Cooldown System
Each tower stores `abilityLastUsed` (timestamp in ms). On ability activation: check `now - abilityLastUsed >= cooldown`. On draw: show cooldown arc as `cdRatio = 1 - elapsed/cooldown`, draw arc from `-PI/2` to `-PI/2 + PI*2*cdRatio`. Alchemist's Catalyst ability, when used, subtracts a percentage from every *other* buffed tower's `abilityLastUsed` rather than resetting its own — implement as `tower.abilityLastUsed -= cooldown * refreshPct`.

### Status Effect Decay
Enemies store `slowFactor` (1.0 = normal, 0.45 = slowed). Each frame: `if (now > slowEnd) slowFactor = min(1.0, slowFactor + dt * 0.001)`. This gives smooth linear recovery over ~1 second after effect expires.
Frozen enemies: `speedMult = freezeEnd > now ? 0 : slowFactor`. Freeze completely overrides slow. Shielded enemies track a separate `shieldHp` pool that damage is deducted from first; once `shieldHp` hits 0, remaining damage overflows into `hp`; `shieldHp` regenerates to full if `now - lastHitTime > 4000`.

### Particle System
```javascript
particles.push({x, y, vx, vy, life, maxLife, color, size});
// update: x += vx; y += vy; life--;
// draw: alpha = life/maxLife; size *= alpha;
```
Keep particles array trimmed: `particles = particles.filter(p => p.life > 0)`. Spawn 4-25 particles per event depending on severity (see updated counts in Visual Effects).

### Screen Shake
Store `shakeTimer` (frames remaining) and `shakeIntensity` (pixel magnitude). In draw(): `ctx.translate((random-0.5)*intensity*2, (random-0.5)*intensity*2)`. Decrement timer each frame. Use `ctx.save()/restore()` to isolate shake from other transforms.

---

## Edge Cases & Gotchas

1. **Audio autoplay policy** — browsers block audio until first user gesture. Always call `ensureAudio()` (which creates the AudioContext) inside a click/touch handler, not in init().
2. **Delta time capping** — if tab is hidden, `dt` can jump to thousands of milliseconds. Cap at 33ms (~30fps minimum) *before* applying `speedMult`, so 3x speed never produces a multi-second physics jump.
3. **Canvas scaling** — when canvas is CSS-scaled (responsive), map mouse coordinates using `rect.width/width` and `rect.height/height` ratios.
4. **localStorage quotas** — keep saved data small (just numbers and small objects). Don't save game state, only prestige/high-score/streak/daily data. Daily-score keys accumulate one per calendar day played — consider pruning entries older than ~60 days on load to avoid unbounded growth.
5. **Enemy filter timing** — filter dead enemies after all updates but before next frame's drawing. Filter projectiles after all shots resolve.
6. **Elemental DoT tick rate** — apply burn damage as `dps * dt / 1000` per frame, not per-second fixed amounts, to maintain consistency across framerates. This applies to Steam Burn, Overcharge explosions, and Firestorm's burning ground identically.
7. **Tower range/damage with upgrades** — always recompute effective range/damage each frame (synergy/set/cascade/Alchemist-aura/Relic bonuses can all change as towers are sold, fused, or Relics picked), don't cache it. This is the single most common source of stale-bonus bugs as the bonus sources multiply in v2.
8. **Combo range vs synergy range** — use slightly different distances: ~80px for combo detection, ~70px for synergy adjacency, ~100px for cascade clustering, and the Alchemist's own `range` stat (90/110/130 by level) for its aura. Don't conflate these constants.
9. **Set bonus scope** — set bonuses apply globally to all towers of that type on the map, not just clustered ones. A player can build a sacrificial 3rd tower elsewhere to activate a set bonus without sacrificing positioning. Fusion can *remove* this activation — always warn before fusing if it would drop a set below threshold.
10. **Tower property naming** — tower objects store level as `lv` (not `level`). Helpers like `tStats`, `upgCost`, `sellVal`, and `actAbility` must reference `t.lv`. Using `t.level` returns `undefined` and produces NaN values for all scaling calculations, silently breaking towers. Ultimate (fused) towers should still set `lv: 3` (capped, non-upgradeable) so these helpers don't need special-casing.
11. **Path segment length assignment** — when building path segments, assign `tl` to the segment object (`pathSegs[i].tl += l`), not to a global variable. A global `pathTL` that shadows the intended property causes `dist >= tl` to evaluate true at spawn (0 >= 0), killing enemies instantly.
12. **Projectile element lookup** — `fireProj` calls `ttEl(tgt.typeIdx)` on the target enemy, but enemies store affinity, not typeIdx. Pass the tower type index separately or resolve element from the tower that fired the projectile.
13. **Cascade detection ordering** — `checkCas(e)` must check Elemental Convergence (all 4 elements) _before_ Arcane Surge (Triple) (arcane + any 2+), otherwise 4-element clusters never trigger Convergence. `checkCas` should now return an array of all matching cascades for proper stacking, and per the new Convergence/Arcane-Surge overlap rule, suppress Arcane Surge's fire-rate bonus specifically when Convergence is also active in the same cluster (avoid double-dipping on the same stat).
14. **Cascade cluster detection** — use connected-component (flood-fill) clustering rather than triple enumeration. Triple-based detection can never find Elemental Convergence because each triple covers at most 3 of the required 4 unique elements. Exclude Alchemist towers from cluster membership entirely (checked via `!towers[i].element`).
15. **Ability upgrade visual distinction** — Lv.3 abilities should have a clearly different visual signature (color flash, extra particles, larger AoE) so the player feels the power spike. Ultimate (fused) tower abilities should look distinctly different again (e.g., a rotating ring flash) so fusion feels like a genuine tier-up rather than "Lv.4."
16. **Basic ability projectile count** — the formula `3 + lv * 2` gives Lv.1=5, Lv.2=7, Lv.3=9 but the intended spec is Lv.1=3, Lv.2=5, Lv.3=7. Use `1 + lv * 2` for correct scaling.
17. **Game speed and audio/particle timing** *(NEW)* — `speedMult` should only scale the `dt` fed into `update()`. Do not scale oscillator durations, particle `life` decrements independent of `dt`, or CSS transition timings, or the game will sound/look broken at 3x. Anything driven by `dt` scales naturally; anything on a fixed real-time timer should not be touched.
18. **Relic stacking edge cases** *(NEW)* — percentage-based Relics (e.g., Ember Core +20% Fire damage) should stack additively with each other and with prestige/set bonuses of the same category inside the central `finalDamage` formula, not multiplicatively chained, or two similar Relics plus a prestige upgrade can produce absurd multipliers. Keep all "+X% damage" sources summed into one bracket before applying the elemental/crit multipliers.
19. **Fusion + set bonus recompute** *(NEW)* — after `fuseTowers()` runs, immediately call the same recompute pass used on sell/placement for synergy, set, cascade, and Alchemist-aura bonuses across the *entire* tower array, not just the new Ultimate tower — removing 3 towers changes everyone else's set counts too.
20. **Daily seed determinism** *(NEW)* — every `Math.random()` call inside wave generation, perk offers, relic offers, and golden-enemy scheduling must be routed through the injectable RNG so a Daily Challenge run is byte-for-byte identical for all players on a given date; leaving even one call on `Math.random()` breaks the fairness guarantee the whole feature depends on.
21. **Kill-streak reset semantics** *(NEW)* — streak should decay gradually over time when no kills occur (not hard-reset), but reset to 0 instantly when a life is lost. Conflating these two paths (e.g., using the same decay function for both) makes losing a life feel less impactful than it should.

---

## Feature Toggle System (Implementation Detail)

The game includes a collapsible **⚙️ Features** panel with 15 independent checkboxes, all default **OFF**. Each feature gates its entire logic chain behind a boolean in the `F` state object. Basic tower is always visible (not gated). When any checkbox is toggled, `applyFeatures()` syncs UI visibility and reloads the page to cleanly apply logic gates.

| Toggle ID | Feature | Gated Logic |
|-----------|---------|-------------|
| `f-sniper` | Sniper Tower | Shooting, targeting, set bonuses, synergy, abilities |
| `f-rapid` | Rapid Tower | Same as above |
| `f-ice` | Ice Tower | Same as above |
| `f-bombardier` | Bombardier Tower | Same as above |
| `f-alchemist` | Alchemist Tower | Same as above, plus aura pass |
| `f-elements` | Elemental System | Enemy affinities, elMult multiplier, cascade detection |
| `f-upgrades` | Tower Upgrades | Upgrade button in panel, Lv.2/Lv.3 scaling |
| `f-abilities` | Active Abilities | Ability bar, cooldown system, keybinds |
| `f-combos` | Combo/Cascade/Sets | Adjacent combos, set bonuses, cascade reactions, consecutive hit tracking |
| `f-enemies` | Special Enemies | Shielded/splitter/healer/teleporter/golden spawns, boss phases, elite/regen types |
| `f-prestige` | Prestige System | Prestige shop, prestige HUD (✨ Lv.), points accrual |
| `f-relics` | Relic System | Relic modal, shard counter, relic effects in damage formula |
| `f-streaks` | Kill Streaks/Overdrive | Streak counter, overdrive timer, streak milestones |
| `f-daily` | Daily Challenges | Daily seed, challenge effect, streak days tracking |
| `f-endless` | Endless Mode | Endless mode button, endless best score |
| `f-fusion` | Tower Fusion | Fuse button in upgrade panel, 3-Lv.3 detection, Ultimate tower creation |
| `f-speed` | Speed Multiplier | All three speed buttons (1x/2x/3x) |
| `f-maps` | Multiple Maps | Map dropdown population (multiple path options) |
| `f-waves` | Wave Cost | Money cost to start waves (`wave * 2`, default OFF = free waves) |

**Critical implementation notes:**
- `applyFeatures()` **must be called at init** (after `loadSave()`) — without this, toggles don't sync on page refresh and UI shows elements that should be hidden.
- Each feature's logic must check its flag **at the top of the relevant code path**, not just in one place. For example, `F.elements` gates enemy affinity assignment in `spawnWave()` AND the `elMult()` call in the tower shooting loop.
- When a feature is disabled, provide fallback objects to prevent crashes: e.g., `{dm:1,rm:1,fr:1}` for synergy when combos are off, `{dm:1,rm:1,fr:1,cr:0,sr:0}` for set mult.
- Relic modal must check `F.relics` at the top of `showRelicPick()` — even if `shards >= 4`, don't show the modal if relics are disabled.
- Prestige HUD in `updateHUD()` must check `F.prestige&&pl>0` — showing it whenever `pl>0` without the flag check causes it to appear even when prestige is disabled.

## Implementation Notes & Bug Fixes (v2.1)

These are lessons learned during implementation that are critical for any LLM or developer building this game from spec:

### CRITICAL BUG: Wave 1 Invincible Enemy
**Symptom:** First enemy of wave 1 appears invincible — HP bar stays full green despite towers shooting and dealing damage.
**Root cause:** ALL enemies were pushed with `dist:0` in `spawnWave()`, causing them to stack on top of each other at the exact same position. The first enemy was visible, but subsequent enemies were hidden behind it. Towers auto-targeted whatever enemy was closest (which could be any enemy in the stack), making it appear that the visible enemy was invincible.
**Fix:** Use cumulative spacing instead of a fixed offset:
```javascript
let cumDist = 0;
for(let i = 0; i < cnt; i++) {
    const spacing = Math.max(36, ((40 - wave * 2) - i * 8) * 3);
    enemies.push({ dist: -cumDist, ... });
    cumDist += spacing;
}
```
This ensures enemies are spread out along the path from spawn. First enemy at dist:0, second at dist:-spacing, etc.

### Enemy Count & Scaling Changes (from spec)
The original spec said `5 + wave * 3` enemies per wave. This was too harsh for early waves:
- **Wave 1:** 8 enemies → changed to **4** (`3 + wave`)
- **Wave 2:** 11 → **5**
- **Wave 3:** 14 → **6**

HP scaling reduced: `(30 + wave * 18)` → `(25 + wave * 12)`
Speed scaling reduced: `(1.2 + min(wave * 0.08, 3))` → `(1.2 + min(wave * 0.06, 2.5))`
Reward scaling reduced: `(5 + floor(wave * 1.5))` → `(4 + floor(wave * 1.2))`

### Progressive Spawn Pacing
Spawn interval is now wave-scaled: `Math.max(600, 1500 - wave * 90)` ms.
- Wave 1: ~1410ms between enemies
- Wave 5: ~1050ms
- Wave 10+: 600ms (capped)

This gives early waves breathing room and late waves increasing pressure.

### Intra-Wave Spacing
Enemies within a single wave are spaced by index:
- Head of wave: wide gaps (~38 units on wave 1)
- Tail of wave: tight clusters (~15 units minimum)
- Formula: `Math.max(36, ((40 - wave * 2) - i * 8) * 3)`
- This creates a natural "head-to-tail" formation that's visually obvious.

### HP Guard (Defensive Programming)
Always guard enemy HP against Infinity/NaN:
```javascript
if (!isFinite(p.target.hp)) p.target.hp = p.target.maxHp || 1;
p.target.hp -= finalDmg; // Simple subtraction works fine after the guard
```
The `Object.defineProperty` approach is unnecessary — simple assignment works. The guard is what matters.

### Tower Property Naming (Already in spec, re-emphasized)
Towers store level as `lv` (not `level`). All helpers must use `t.lv`. Using `t.level` returns `undefined` → NaN stats → silent tower breakage.

### Path Segment Length (Already in spec, re-emphasized)
Assign `tl` to the segment object: `pathSegs[i].tl += l`. Never use a global `pathTL` variable — it shadows the property and causes enemies to think they've reached the end at spawn (0 >= 0).

### Cascade Detection (Already in spec, re-emphasized)
Must use connected-component (flood-fill) clustering. Triple enumeration can never find 4-element Convergence. Check Convergence BEFORE Arcane Surge.

### Ability Projectile Count
Basic tower Arcane Blast: `1 + lv * 2` gives Lv.1=3, Lv.2=5, Lv.3=7. NOT `3 + lv * 2` (which gives 5/7/9).

## What to Skip (YAGNI)
- Multiple currencies beyond money + prestige points + perfect-wave shards (which merely gate a single bonus Relic pick) — don't add further currencies
- Tower placement preview ghost — crosshair + status text is sufficient
- Minimap — canvas is small enough to see everything
- Sound volume controls — keep it simple; volume is low by default
- Real multiplayer, server-side leaderboards, or Relic/save trading between players — `localStorage` high scores and the Daily Challenge's date-seeded fairness are the replay hooks; no backend required
- Server-side validation of Daily Challenge scores — trust the client, consistent with the rest of the persistence model
- Achievements/badges system — the Relic tray, streak banners, and prestige shop already cover the "visible progress" itch; adding a parallel achievements list would be redundant scope

---

## Success Criteria
The game should feel **immediately playable** and **deeply strategic**, with layered hooks for short, medium, and long-term retention:
1. **First 30 seconds:** Player places towers, starts wave 1, sees enemies die, earns money, notices the kill-streak counter ticking up. Feels good.
2. **After wave 5:** Player understands elemental counters, synergy placement, ability timing, set bonuses, and has picked their first Relic — a build identity starts forming.
3. **After wave 15:** Player is building optimized tower compositions with combo chains, elemental cascades (now with real effects), Alchemist support placement, and is timing abilities around boss phases and Golden Enemy windows.
4. **After wave 20+:** Player fuses their first Ultimate tower, or pushes into Endless Mode to see how far the build can go.
5. **After game over:** Player sees prestige points earned, buys permanent upgrades (including Momentum and Relic Seeker), and wants to try again with a different Relic-driven build.
6. **The next day:** Player is drawn back by the Daily Challenge and streak counter — a low-friction reason to return that doesn't require finishing a full run.
7. **Any time:** Pressing Q/E/R/F/T/Y feels satisfying. A crit or a streak milestone lands with a distinct, punchy hit. Visual feedback is instant and clear.

The single file should be **under 99KB** (actual: ~74KB with all v2 systems implemented) and load with no console errors. Works in Chrome, Firefox, Safari, Edge. Mobile browsers work but aren't the primary target (touch controls would need extra work).
