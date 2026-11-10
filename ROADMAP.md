# 521 Studios Roadmap

Direction is set by Devon, and pm-521-studios keeps this file current.
Progress is tracked in beads (`bd_521Studios`). This file holds only the why, the milestones and the order.

## Why

lets-roll started during covid as a simple dice roller used next to Miro.
It began with PF2, then also served PF1 and 5e, and grew features added by hand on request.
An earlier encounter builder attempt mostly failed, while the pfsrd2 data kept growing.
The app has moved from Heroku to AWS.
It is now being broken into small unix-style services and apps, replacing the lets-roll monolith.

Devon is in the last chapter of Abomination Vaults.
He ran treasure with physical coins and item cards. Players enjoy them, but they add cognitive load, and the treasure history lives in Google Sheets.

## North star

Give Claude a module, and it builds the encounters through the API with treasure attached, so prep is mostly done before the session.
The GM then runs the game with every monster and item at hand, without dozens of AoN pages open.
In a module, treasure implies encounters.

## Milestones

| | Epic | Scope |
|---|---|---|
| M1 | `bd_521Studios-s9rv` | **Treasure loop.** Encounter Builder release → party-treasure, then give, sell, identify, and who-has-what. Includes `uotk`. |
| M2 | `bd_521Studios-ivqk` | **Agent authoring.** Non-browser GM auth for an agent, a module → encounters skill, and a trial on one real module. Includes `gql1`. |
| M3 | `bd_521Studios-we5k` | **GM run-time reference view.** Monsters, items and treasure state in one place, built on pfsrd2-display. |
| M4 | `bd_521Studios-fcu0` | **Prod cut-over** before the next campaign, 1–3 months from 2026-11-10. Includes `pmib` and `chng`. |
| Later | | Decomposing lets-roll, 521-engine (conditions and templates), a quick tool that needs no sign-in, wyrd. |

## Current order

1. Finish the chapter maps (`bd_521Studios-muz0`).
2. Then the treasure loop (M1).
