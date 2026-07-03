# AGENTS.md

Guidance for AI agents (and humans) working on this repo. Read this before editing anything.

## What this is

A tower defense game. The entire product is **one self-contained file**: `tower_defense.html`. No build step, no frameworks, no npm, no CDN, no backend. Everything — HTML, CSS, JS — lives inline in that file. `tower_defense_spec.md` is the full design spec (the "what it should be"); the `.html` is the implementation (the "what it currently is"). The two have drifted — see *Spec drift* below.

To "run" the game: open `tower_defense.html` in a browser. To "test": play it. There is no test suite.

## File map

- `tower_defense.html` — the game. ~770 lines, single `<script>` block at the bottom.
- `tower_defense_spec.md` — authoritative design doc. Read the *Implementation Notes & Bug Fixes (v2.1)* and *What to Skip (YAGNI)* sections before changing behavior; they encode hard-won footguns.
- `README.md`, `LICENSE` — repo metadata.

## Architecture inside the HTML

The `<script>` block has a fixed layout. Get oriented by these regions:

1. **Constants** — `CW/CH` (canvas 800×600), `TT` (tower-type table), `EC` (element→color), `CR/SR/PM/TMD` (placement/spacing thresholds).
2. **State globals** — a flat list of `let` vars (money, lives, wave, `towers`, `enemies`, `projs`, `relics`, `prUpg`, daily-challenge state, etc.). There is no module/encapsulation; state is global and mutated in place.
3. **Persistence** — `loadSave()` / `saveBest()` / `savePrest()`. All `localStorage` keys are prefixed `td_` (see *Persistence keys*).
4. **Core loop** — `posAt(dist, segmentIndex)` → path math; `spawnWave()`; `update(dt, now)` (physics); `draw()` (rendering); `updateHUD()` / `updateComboPanel()` (DOM).
5. **Feature toggle system** — the `⚙️ Features` panel. See its own section below.

### Non-negotiable conventions

These are called out in the spec as silent-failure traps. Honor them:

- **Delta-time only.** All movement scales by `dt`. `speedMult` (1×/2×/3×) multiplies the `dt` passed into `update()` **only**. Never feed `speedMult` into audio oscillators, particle `life`, or CSS transitions — those run on real time and break at 3×.
- **Tower level is `t.lv`, never `t.level`.** `t.level` is `undefined` → NaN stats → silent tower breakage.
- **`posAt(dist, si)`** positions everything on the path. Segment lengths must live on the segment object (`pathSegs[i].tl`), never a shadowing global, or enemies think they've finished at spawn.
- **Enemy spawn spacing is cumulative** (`dist: -cumDist`), not a flat offset. A flat offset caused the original "wave-1 invincible enemy" bug (all enemies stacked at one point; towers targeted the wrong one).
- **Guard enemy HP against Infinity/NaN** before subtracting damage. Simple assignment is fine; the guard is what matters.
- **Daily Challenge determinism:** every `Math.random()` in wave/perk/relic/golden-enemy generation must route through the injectable RNG. One stray `Math.random()` breaks the fairness guarantee that the whole Daily feature depends on.
- **Damage stacking:** all "+X% damage" sources (synergy, set, Alchemist aura, relics, prestige) sum into one additive bracket inside `finalDamage` *before* elemental/crit multipliers. Do not chain them multiplicatively.
- **Kill streak:** decays gradually when idle, but resets to 0 instantly on life loss. Two different paths, do not share one function.
- **Recompute bonuses globally** on place / sell / fuse — synergy, sets, cascades, and aura all depend on the full tower array, not just the touched tower.

### Feature toggle system (important)

A collapsible **⚙️ Features** panel exposes ~15 independent checkboxes (`f-*` IDs), all **OFF** by default. State lives in the `F` object. Toggling any calls `applyFeatures()`, which syncs UI visibility and **reloads the page** to apply logic gates cleanly.

- `applyFeatures()` **must** run at init after `loadSave()`, or toggles desync on refresh.
- Each feature's logic must check its flag **at the top of every relevant code path** (e.g. `F.elements` gates *both* affinity assignment in `spawnWave()` *and* the `elMult()` call in the shooting loop), not just in one spot.
- Disabled features need fallback objects so the math doesn't crash (e.g. `{dm:1,rm:1,fr:1}` for synergy when combos are off).
- Basic tower is always visible and never gated.

## Persistence keys (`localStorage`, all `try/catch` guarded)

| Key | Meaning |
|-----|---------|
| `td_best` | Best campaign score |
| `td_pp`, `td_pl`, `td_pu` | Prestige points, level, upgrades (JSON) |
| `td_endless_best`, `td_endless_unlocked` | Endless mode high score + unlock flag |
| `td_relics` | Relic inventory (JSON array) |
| `td_streak_days`, `td_last_play_date` | Daily Challenge streak |
| `td_daily_seed`, `td_daily_completed` | Current daily seed + completion flag |

When adding a new persisted value: add it to `loadSave()`, `savePrest()` (or a new saver), and reset it in whatever "new run" / "restart" path applies. Never assume the key exists.

## Spec drift (implementation ≠ spec)

The HTML has diverged from `tower_defense_spec.md`. The HTML is source of truth for *current behavior*. Notable divergences an agent will hit:

- **Alchemist** has element `fire` and ability **Toxic Cloud** (`G`) in code — the spec says element *None* and ability *Catalyst*. Mind this if you touch Alchemist/cascade logic.
- **Maps:** spec lists 3 maps; the `map-select` dropdown has ~12 (`MAPS` array).
- **Relic pool** (`RLD`) is a different set of relics than the spec's example table (Phoenix Feather, Dragon Scale, Frost Crystal, Storm Core, Ember Stone, Void Shard). Don't "fix" them to match the spec without checking what the code actually consumes.
- **Prestige upgrade IDs** (`richStart`, `powerBoost`, `fortify`, `bulkBuy`, `swiftSouls`, `alliedSpirit`, `quickCast`) — use these IDs, not the spec's prose names.

When in doubt about current behavior, **grep the HTML**, don't trust the spec's tables.

## How to make changes

- Edit `tower_defense.html` directly. No transpile, no format step. Keep the minified-ish one-line style of existing functions when extending them so diffs stay reviewable.
- Verify by reloading the file in a browser. For logic that depends on progression (relics, prestige, endless, daily), clear the relevant `td_*` `localStorage` keys to test the fresh-state path too.
- No backend, no network — do not add `fetch`, CDNs, or external assets. The zero-dependency constraint is a hard requirement.
- Keep everything in the single file. Do not split into modules.

## What NOT to add (YAGNI — from spec)

No extra currencies beyond money + prestige points + perfect-wave shards. No placement-preview ghost (crosshair + status text is enough). No minimap. No volume controls. No multiplayer, leaderboards, or server-side anything. No achievements/badges system (relics, streaks, and prestige already cover visible progress). `localStorage` + date-seeded Daily fairness are the only persistence/replay hooks.
