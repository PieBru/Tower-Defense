# Tower Defense

A single-file tower defense game built with vanilla HTML, CSS, and JavaScript — zero external dependencies. Just open the file and play.
I made it to have fun while entertaining my nephews on both playing and designing such a software, and to have an empirical benchmark platform measure my fully-local LLM inferencing, software architecting, coding capabilities.

## Features

- **Self-contained** — everything in one `tower_defense.html`, no build step, no frameworks, no CDN.
- **Canvas rendering** at 800×600 with delta-time physics (runs identically at 30–144 fps).
- **Multiple tower types** (Basic, Sniper, Rapid, Ice, Bombardier, Alchemist support tower) with fusion into Ultimate towers.
- **Multiple maps**: Winding Path, Serpent, Crossroads, and more.
- **Roguelite run layer**: per-wave perks, every-5-wave Relics, kill-streak/crit juice, set bonuses and elemental cascades.
- **Special enemies & boss phases**, plus Endless Mode after clearing the campaign.
- **Daily Challenge** with seeded RNG and streak tracking.
- **Web Audio API** synthesized sound effects; **localStorage** persistence for best scores, prestige, and streaks.
- **Game speed multiplier** (1×/2×/3×) that scales `dt` while rendering at real time.

## Play

Open `tower_defense.html` in any modern browser.

## Spec

Full design spec: [`tower_defense_spec.md`](tower_defense_spec.md).

## License

Licensed under the [Apache License 2.0](LICENSE).
