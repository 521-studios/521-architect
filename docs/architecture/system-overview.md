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

Encounter Builder and Party Treasure are shipped to staging. Their prod
cut-over is bead `pmib` (open, Devon-gated).

**lets-roll as identity provider.** lets-roll is the cluster's OIDC provider
(Doorkeeper, Authorization Code + PKCE, RS256 JWT, JWKS). It also serves a
small `/api/v1` with Me and Games reads, which the sibling APIs use to decide
GM vs member. It first shipped to prod on 2026-11-10, with clients
registered at deploy time for encounter-builder-web and party-treasure-web.
There is no `end_session_endpoint`, so SPA sign-out clears only local tokens
(`bd_521Studios-2l4x`, P3).

**lets-roll owns live combat.** HP, conditions and initiative belong to
lets-roll. Encounter Builder is prep-only (Devon, 2026-11-10).

**Encounter Builder** — GM prep: campaigns, chapters, encounters, monsters, treasure
- `encounter-builder-api`: Go Lambda behind `/api/app/*`, DynamoDB
  (`CAMPAIGN#<game_id>`). Owns encounters, chapters, the GM gate, chapter map
  images (muz0) and the release push to party-treasure.
- `encounter-builder-web`: Vite + React GM SPA, a thin consumer of
  pfsrd2-display.
- Design: [encounter-treasure-cluster.md](encounter-treasure-cluster.md),
  [encounter-builder-v2.md](../design/encounter-builder-v2.md)

**Party Treasure** — the party's held loot after release
- `party-treasure-api`: Go Lambda behind `/api/party/*`, DynamoDB
  (`PARTY#<game_id>`). Owns items (held/sold/assigned, masking, identify DC,
  history), the coin ledger, trades and party settings. It enforces masking
  server-side: non-GMs see `mask_label`, and the ref is blanked. It never
  calls pfsrd2-data-api, and stores ContentRefs opaquely.
- `party-treasure-web`: player and GM SPA. It renders items with
  pfsrd2-display's `ItemCard`, masked ones included.

**wyrd** — random generation for TTRPGs
- Generator plugins. The first is `kenning`, which makes British Isles place
  names with etymology. API: `GET /api/manifest`, `GET|POST /api/<generator>`.
  Every response uses the envelope `{result, explanation, components, seed}`.
- Flask on Lambda (Function URL), reading a bundled SQLite runtime DB, plus a
  Svelte SPA. Its Terraform lives in `wyrd/terraform` for now; it is to be
  folded into `infra/` later (`bd_521Studios-t65j`, P3). It keeps its own beads tracker (intentional).
- Offline authoring: a lexicon mined from place-name dictionaries and
  wiktextract. Ollama on Devon's MacBook is used at authoring time only,
  never on the request path.
- No other 521 service consumes it yet. encounter-builder-web may later use
  it for names.

**pfsrd2-display** — React library for stat blocks, item cards and search
- THE library for displaying and selecting PF2 objects (library-first rule,
  [encounter-builder-v2.md](../design/encounter-builder-v2.md))
- Schema-version-aware (V1.2, V1.3/1.4)
- Change highlighting via JSON Pointer paths
- Consumers: encounter-builder-web, party-treasure-web. Its own harness is
  deployed at `display.pfsrd2.staging.521studios.com`. Not yet re-integrated
  into lets-roll.
- Repo: `521-studios/pfsrd2-display`

### In Design

**521-engine** — DAG-based tabletop RPG game engine
- Design docs only; code is on hold. The session owns its design docs, and
  521-architect reviews where the engine crosses system boundaries.
- Decided (Devon, 2026-11-10): stacking rules are data. GM-authored content is
  declarative only. Vetted compute is compiled at build time and referenced
  by content-hashed id, with no runtime eval. Template stacking
  (pfsrd2-data-api `4zx0`) waits for engine Phase 5. See
  [content-ref.md](content-ref.md).
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

Human owner, then the agent session that stewards the repo day to day. The
live roster is `~/521Studios/TEAM.md`, maintained by `pm-521-studios`. If it
and this table disagree, trust TEAM.md.

| Service | Owner | Steward session | Repo |
|---------|-------|-----------------|------|
| lets-roll | Morgan + Devon | lets-roll | 521-studios/lets-roll |
| encounter-builder-api | Devon | encounter-builder-api | encounter-builder-api |
| encounter-builder-web | Devon | encounter-builder-web | encounter-builder-web |
| party-treasure-api | Devon | party-treasure-api-cf | party-treasure-api |
| party-treasure-web | Devon | party-treasure-web-ee | party-treasure-web |
| pfsrd2-data-api | Devon | pfsrd2-data-api | 521-studios/pfsrd2-data-api |
| pfsrd2-display | Devon | pfsrd2-display | 521-studios/pfsrd2-display |
| 521-engine | Devon | 521-engine | 521-studios/521-engine |
| wyrd | Devon | wyrd | wyrd |
| 521-architect | Devon | 521-architect | 521-architect |
| pfsrd2-parser | Devon | pm-masterwork-tools (session `pfsrd2`) | devonjones/PFSRD2-Parser |
| pfsrd2-data | Devon | pm-masterwork-tools (session `pfsrd2`) | devonjones/PFSRD2-Data |
| pfsrd2-web | Devon | pm-masterwork-tools (session `pfsrd2`) | devonjones/PFSRD2-Web |
| pfsrd2-corpus | Devon | pm-masterwork-tools (session `corpus`) | pfsrd2-corpus |
| pfsrd2-automation | Devon | none yet | 521-studios/pfsrd2-automation |
| infra | Devon | 521-infra | 521-studios/infra |
| infra-frontend | Devon | 521-infra-frontend | 521-studios/infra-frontend |
| tools | Devon | none yet | 521-studios/tools |
| smoke_tests | Devon | none yet | 521-studios/smoke_tests |

Coordinate on the pfsrd2-parser/data/web/corpus repos through
`pm-masterwork-tools`, a separate PM.

## Agent Fleet

`pm-521-studios` is the project manager. It relays to and from Devon and owns
the `bd_521Studios` beads store. Each worker session stewards the repo listed
above. Questions and status go to the PM, and workers may talk to each other
directly.
