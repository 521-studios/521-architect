# 521 Studios Roadmap

Direction is set by Devon, and pm-521-studios keeps this file current.
Progress is tracked in beads (`bd_521Studios`; bare IDs below omit that prefix). This file holds only the why, the milestones and the order.

## Why

lets-roll started during covid as a simple dice roller used next to Miro.
It began with PF2, then also served PF1 and 5e, and grew features added by hand on request.
An earlier encounter builder attempt mostly failed, while the pfsrd2 data kept growing.
The app has moved from Heroku to AWS.
It is now being broken into small unix-style services and apps, replacing the lets-roll monolith.

Devon is in the last chapter of Abomination Vaults.
He ran treasure with physical coins and item cards. Players enjoy them, but they add cognitive load, and the treasure history lives in Google Sheets.

## Context

- The audience is small: about a dozen players in Devon's group, plus trusted collaborators.
- No revenue intent is recorded.
- The next campaign starts 1–3 months from 2026-11-10, which sets M4's deadline.

## North star

Give Claude a module, and it builds the encounters through the API with treasure attached, so prep is mostly done before the session.
The GM then runs the game with every monster and item at hand, without dozens of AoN pages open.
In a module, treasure implies encounters.

## Milestones

| | Epic | Scope |
|---|---|---|
| M1 | `s9rv` | **Treasure loop.** Encounter Builder release → party-treasure, then give, sell, identify, and who-has-what. Includes `uotk`, item modification (`yyt3`), and precious-material and staff pricing (`76bn`). The release is the last step of the encounter's treasure package flow (`we5k.2.5`), not a standalone button. |
| M2 | `ivqk` | **Agent authoring.** Non-browser GM auth for an agent, a module → encounters skill, and a trial on one real module. Includes `gql1` and the fidelity list below. |
| M3 | `we5k` | **GM run-time reference view.** Monsters, items and treasure state in one place, built on pfsrd2-display. Includes `we5k.1` and the encounter lifecycle (`we5k.2`). |
| M4 | `fcu0` | **Prod cut-over** before the next campaign. Includes `pmib`, `chng` and the security gate below. |

**M2 fidelity list** (gaps a real module exposes): `d886` conditional, state-gated and repeatable encounters (the most impactful), `n4ol` fight-or-social resolution branches, `0o77` modifiable creature equipment, `77l2` creature equipment in the treasure total, `kdv8` creatures customised beyond templates, `hzg8` per-instance identity and initiative, `ef41` and `agle` hazards, haunts, traps and afflictions as stat entities, `28se` destructible objects, `vwum` typed exit edges, `72wg` game_id collisions in skills.

**M4 security gate** (closed before prod): `swkv` ECS role reads any secret, `cds4` staging GM password in git history, `wti5` a `--type` build wipes the prod index, `bwbz` stray directories indexed into prod S3, `fcu0.1` plaintext admin token in fleet-reference.md, and the player privacy leaks `lets-roll-7aj`, `lets-roll-cd1` and `lets-roll-yaa`.

### Encounter lifecycle (M3, `we5k.2`)

An encounter moves from prepared, to executing in lets-roll (pulled by lets-roll or pushed by EB), to complete.
EB owns the lifecycle, the kill state and the treasure package; lets-roll owns live combat, and only kills and the outcome flow back.
Both directions are server-to-server.
For conflicts, the latest change wins and EB is the record (confirmed by Devon).
Design: [encounter-lifecycle-bridge.md](docs/architecture/encounter-lifecycle-bridge.md). lets-roll prerequisite: `lets-roll-jlm`.

## Per-app must-haves

Extend this as Devon names more.

**Encounter Builder**
- Each encounter gets a room view: a crop of the chapter map showing the room and the area around it, so you can see how it fits in. An optional upload for the encounter overrides the crop. (`muz0.7`)
- From an encounter, move to its neighbours along the exits graph, or click out to the full chapter map. (`muz0.8`)
- Encounters record progress during play: status, defeated monsters, treasure released or taken, and GM notes. (`we5k.1`, under M3)
- Show the canonical 4-PC difficulty band next to the party as configured. (`529w`, shipped)
- Treasure comes in pools with gates.
- Creature equipment can be modified. (`0o77`)

**Party Treasure**
- Identify an unidentified item. (`uotk.3`)
- Trades between players.
- A split calculator: direct assignment or shares.
- Mark an item for sale; sales pay the GM's percent, and pure treasure (gems, art) sells at 100%.
- Masking is enforced server-side: players see only the mask label until an item is identified.
- Item history and the coin ledger are append-only.

**pfsrd2-display**
- Displays read like the books and AoN, with variants stacked rather than in a dropdown. One exception Devon chose: the creature description stays below the stat block.

**521-engine**
- `explain()` on every computed value.

**Quick tool (no sign-in)**
- Land straight in an encounter, with live difficulty and XP. (`vf84`)

## Cross-app UX laws

These apply to every app. The full text lives in TEAM.md rule 8 and in the PM's memory notes (`feedback_encounter_builder_ux`, `feedback_book_faithful_display`, `project_encounter_ui_restructure`), so it isn't copied here.

## Decisions

- Encounter Builder covers prep plus progress, and lets-roll runs live combat. This revises the earlier "Encounter Builder is prep-only" decision.

## Current order

1. Finish the chapter maps (`muz0`).
2. Then the treasure loop (M1).

## Later

- Decomposing lets-roll.
- 521-engine: conditions and templates.
- Monster Lab and Treasure Lab.
- The Quantum Dice Service. A design doc exists at `~/521Studios/quantum-dice-fairness-system.md`; there is no bead yet.
- Open-sourcing pfsrd2-data-api and pfsrd2-display.
- The quick no-sign-in tool as a separate product for a different audience.
- wyrd.
