# Service Extraction Roadmap

## Philosophy

lets-roll started as a monolith: a collaborative dice roller built during COVID. As the product scope has grown (monsters, encounters, character management), pieces are being extracted into independent services.

The extraction principle: **a service earns its independence when it serves more than one consumer.** We don't extract for the sake of microservices. We extract when a capability is needed by multiple applications or when the bounded context is clear enough that independent development and deployment provide real value.

Each extracted service should be useful standalone. The dice roller isn't just "the thing lets-roll uses to roll dice" — it's a quantum entropy dice service that any application could consume.

## Completed Extractions

### pfsrd2-data-api

**Extracted from:** Previously, lets-roll stored monster data. Now it's a standalone Go Lambda API.

**Status:** Production, serving lets-roll's frontend.

**Consumers:** lets-roll (client-side), encounter-builder-web, pfsrd2-display (dev harness). Planned: Monster Lab, open-source public API.

**Boundary:** Owns all Pathfinder 2e structured game data. Read-only. Stateless (SQLite from S3). No user data, no game state.

### pfsrd2-display

**Extracted from:** lets-roll's monster stat block rendering components.

**Status:** Extracted and packaged as `@521studios/pfsrd2-display`. Not yet re-integrated into lets-roll.

**Consumers:** encounter-builder-web, party-treasure-web, and its own staging harness. Planned: replace lets-roll's built-in rendering, then open-source.

**Boundary:** Pure presentation. Takes creature JSON + optional patches, renders stat blocks. No data fetching, no state management. Callbacks (`onRoll`, `onLoadMonster`) let consumers handle interactions.

### Party Treasure (was "Equipment Tracker")

**Built as:** `party-treasure-api` + `party-treasure-web`, running in staging. Prod cut-over is `chng`, under M4.

**Consumers:** encounter-builder-api (release push), and players and GMs via the SPA.

**Boundary:** The party's held loot: items, masking, identify, trades, the coin ledger and sales. It stores ContentRefs opaquely and never calls pfsrd2-data-api. It doesn't know about combat, dice or sessions. See [encounter-treasure-cluster.md](encounter-treasure-cluster.md).

## In Progress

### Monster Lab

**Owner:** Devon

**What:** A tool for customizing existing monsters or creating new ones from scratch. Uses the template engine and eventually the 521-engine for dynamic stat resolution.

**Service boundary:** Monster customization and creation. Consumes pfsrd2-data-api for base creature data. Will produce game objects that can be used in lets-roll encounters.

**Why extract:** Monster creation is useful beyond lets-roll — GMs prep encounters outside of game sessions, content creators publish custom monsters, the workflow is fundamentally different from "rolling dice in a live session."

## Planned

### Quantum Dice Service

**Currently:** Built into lets-roll as the `quantum_rng` gem. Uses ANU Quantum Random Numbers Server API with SecureRandom fallback.

**Extraction plan:** Standalone service that provides verifiably random dice rolls. Could serve lets-roll, Monster Lab (for random encounter generation), or any third-party consumer.

**Service boundary:** Random number generation with quantum entropy. No game logic, no character knowledge. Takes a dice formula, returns results with entropy source metadata.

### 521-engine Integration

**Currently:** In design phase. Pure JavaScript DAG engine.

**Integration path:**
1. Core engine ships as standalone JS package
2. Embedded in pfsrd2-data-api (Go + JS runtime) for server-side stat resolution
3. Embedded in lets-roll frontend for client-side dynamic state (conditions, spells, buffs)
4. Powers Monster Lab's customization UI
5. Eventually powers a character creator

**Not a service extraction** — this is a shared library, not a standalone service. It runs wherever game state needs to be resolved (browser, Lambda, CLI).

## Extraction Checklist

When evaluating whether to extract a capability:

1. **Does it serve more than one consumer?** If only lets-roll needs it, keep it in the monolith.
2. **Is the bounded context clear?** Can you draw a clean line around what it owns and what it doesn't?
3. **Can it be useful standalone?** Would someone use this without lets-roll?
4. **Does independent deployment provide value?** Can it ship on a different cadence than lets-roll?
5. **Is the API contract stable enough?** Extracting something that's still changing shape creates coupling pain.

## Anti-Patterns to Avoid

- **Extracting too early.** If the interface is still being discovered, keep it in the monolith until it stabilizes. pfsrd2-display was extracted after the rendering was mature.
- **Distributed monolith.** If two services can't be deployed independently, they're not really separate services. Shared databases, synchronous call chains, and lock-step deploys are red flags.
- **Extracting for technology reasons.** "This should be in Go" is not a reason to extract. "This serves three consumers and needs independent scaling" is.
- **Breaking the opt-in principle.** An extracted service should not force complexity on consumers that don't need it. The dice service should work with `POST /roll {"formula": "1d20+5"}` — you shouldn't need to understand quantum entropy to roll dice.
