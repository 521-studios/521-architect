# 521 Studios — System Overview

## What We Build

521 Studios builds focused, composable tools for tabletop RPG players — primarily Pathfinder 2nd Edition. The core product philosophy is **opt-in complexity**: every tool starts simple and lets users add depth when they want it, rather than forcing a steep learning curve upfront.

This is deliberately not a monolithic virtual tabletop (VTT) like Roll20 or Foundry. Instead, it's a suite of sharp, independent tools that compose well together. A user who just wants to roll dice gets a dice roller. A user who wants to run full encounters with stat blocks, conditions, and loot tracking can opt into those features without switching platforms.

## Product Suite

### Shipping

**Let's Roll** — Collaborative dice roller and game session manager
- The flagship product. Rails 7.1 backend, React 18 frontend
- Real-time multiplayer via Action Cable (WebSockets)
- Quantum entropy dice via ANU API (with SecureRandom fallback)
- Pixels digital dice integration (Bluetooth physical dice)
- Character management, shortcuts, variables
- Encounter management with PF2e monster stat blocks
- Impersonation system for support staff
- Currently a monolith; services being extracted over time
- Repo: `521-studios/lets-roll`

**pfsrd2-data-api** — Pathfinder 2e structured data API
- Go Lambda service behind CloudFront
- Full-text search (FTS5), typeahead suggestions, edition-aware cross-referencing
- Monster template engine (RFC 6902 JSON Patch for Elite/Weak/etc.)
- SQLite DB downloaded from S3 at cold start, refreshed hourly via ETag
- Serves JSON entries, images, and template-applied creatures
- Planned: open-source release
- Repo: `521-studios/pfsrd2-data-api`

### Extracted, Not Yet Integrated

**pfsrd2-display** — React component library for PF2e stat blocks
- Extracted from lets-roll's monster rendering
- Schema-version-aware (V1.2, V1.3/1.4)
- Template change highlighting via JSON Pointer paths
- `onRoll` and `onLoadMonster` callbacks for consumer integration
- Planned: replace lets-roll's built-in rendering, then open-source
- Repo: `521-studios/pfsrd2-display`

### In Design

**521-engine** — DAG-based tabletop RPG game engine
- Pure JavaScript, zero dependencies
- Resolves game state via directed acyclic graph with topological sort
- `explain()` provides full audit trail for every computed value
- Opt-in complexity: flat numbers → partial decomposition → full DAG
- System-agnostic — PF2e rules loaded as game objects, not hardcoded
- Target: character creation, Monster Lab, dynamic condition/spell tracking
- Repo: `521-studios/521-engine`

### Planned Extractions

**Monster Lab** — Customize/create monsters (Devon, in progress)

**Quantum Dice Service** — Extract the ANU quantum entropy roller as a standalone service

**Equipment Tracker** — Encounter loot drops, party inventory, sell/assign to players

## Data Pipeline

The structured game data that powers the API and display components comes from a dedicated pipeline. This is Devon's domain — the data repos predate 521 Studios and feed into the product ecosystem.

See: [Data Pipeline Architecture](../pipeline/data-pipeline.md)

## People

**Devon Jones** — Data pipeline, game engine, infrastructure, pfsrd2-* ecosystem. Long history with Pathfinder reference tools (Pathfinder Open Reference, 500k+ installs on Android).

**Morgan** — Built lets-roll. Product development and the Rails/React application layer.

## Service Ownership

| Service | Owner | Repo | Org |
|---------|-------|------|-----|
| lets-roll | Morgan + Devon | 521-studios/lets-roll | 521-studios |
| pfsrd2-data-api | Devon | 521-studios/pfsrd2-data-api | 521-studios |
| pfsrd2-display | Devon | 521-studios/pfsrd2-display | 521-studios |
| 521-engine | Devon | 521-studios/521-engine | 521-studios |
| pfsrd2-parser | Devon | devonjones/PFSRD2-Parser | personal |
| pfsrd2-data | Devon | devonjones/PFSRD2-Data | personal |
| pfsrd2-web | Devon | devonjones/PFSRD2-Web | MasterworkTools |
| pfsrd2-automation | Devon | 521-studios/pfsrd2-automation | 521-studios |
| infra | Devon | 521-studios/infra | 521-studios |
| infra-frontend | Devon | 521-studios/infra-frontend | 521-studios |
| tools | Devon | 521-studios/tools | 521-studios |
| smoke_tests | Devon | 521-studios/smoke_tests | 521-studios |

## Agent Fleet

Servitor agents (Pathfinder deity naming convention):

| Agent | Role | Repo |
|-------|------|------|
| Iomedae | Fleet commander | servitor |
| Nethys | API guardian | pfsrd2-data-api |
| Abadar | System architect | 521-architect |
