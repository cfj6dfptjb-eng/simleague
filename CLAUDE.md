# NHL Hockey Simulation

This project builds a realistic NHL simulation: game engine with full play-by-play and stats, player and staff ratings with human factors, 84-game seasons and playoffs, front office (contracts, trades, draft, waivers, minor leagues), owner mode, and team and super admin consoles.

**The full design spec is `docs/SPEC.md`. It is the source of truth.** Read the relevant sections before working on any feature. Don't load the whole file when you only need part of it; its section numbers are stable and cross-referenced.

## How to work in this repo

- **Build in phases** (SPEC section 22.1). Don't start a phase until the previous one passes its tests and its realism checks. Track the current phase in `docs/PROGRESS.md`.
- At the start of each phase, write down in `docs/PROGRESS.md`:
  - the spec sections the phase covers
  - the default values you will use where the spec gives none
  - the realism targets you will check against
- **No hard-coded tunables.** Every rate, weight, threshold and rule value lives in config with a default, a min/max range and a short description (SPEC ground rules, section 18.1). The engine only reads config.
- **Single source of truth:** each rule lives in one module, matching the spec's one-section-per-rule structure.
- **Determinism:** all randomness goes through a seeded RNG passed in explicitly. Same seed and config must give an identical event log.
- **Stats come from the event log only** (SPEC 9.4). Never track a stat separately.
- **Tests alongside code** (SPEC 20.4):
  - `tests/rules/`: every league, game and contract rule
  - `tests/properties/`: invariants, e.g. never fewer than 3 skaters, stats reconcile with the log, cap legality
  - `tests/exploits/`: "bad owner" strategies that must fail
- **Realism report:** after any change to the engine or config defaults, run the validation batch and compare to targets (SPEC 10, 9.6, 20.1). Don't accept a change that pushes a metric out of range without noting why.
- Values in SPEC section 21 (verify list) come from the current NHL rules and may be outdated. Keep them in config with a `source` and `as_of` note. Don't present them as confirmed.

## Tech defaults (change if the owner says otherwise)
- Python 3.11+, standard library plus numpy. pytest for tests.
- Config in YAML or JSON under `nhl_sim/presets/`, loaded into typed dataclasses.
- Consoles and the live viewer come in later phases (6–7). Pick the web stack then and record the choice here.

## Licensing
Real NHL team and player names are for private use only. Keep them in an optional data pack. The default preset for any public build is a fictional league (SPEC 22.4).
