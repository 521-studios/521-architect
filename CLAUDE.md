# 521-architect — CLAUDE.md

This session is `521-architect`, the 521 Studios system architect: it owns
cross-system design and keeps the record of how the services connect.

## What This Is

521-architect is the system architecture repository for 521 Studios. It contains:
- **Architecture documentation** — system overview, service dependencies, extraction roadmap
- **Pipeline documentation** — the pfsrd2 data pipeline from HTML to production API
- **Infrastructure documentation** — AWS infrastructure, Terraform layers, deployment order
- **Engineering standards** — cross-repo conventions, per-repo tooling, security standards

This repo does not contain application code. It contains the architectural record — the map of how all 521 Studios services relate to each other.

## Repository Structure

```
521-architect/
├── CLAUDE.md
├── README.md
└── docs/
    ├── architecture/
    │   ├── system-overview.md             # Product suite, service catalog, stewards
    │   ├── service-dependencies.md        # Runtime + build-time dependency map
    │   ├── content-ref.md                 # ContentRef / modifications[] contract (ACCEPTED)
    │   ├── encounter-treasure-cluster.md  # Encounter Builder + Party Treasure plan
    │   ├── extraction-roadmap.md          # Monolith decomposition plan and principles
    │   └── external-docs-index.md         # Architecture docs in other repos
    ├── design/encounter-builder-v2.md     # Encounter Builder v2 design of record
    ├── goals/encounter-builder-usable.md  # Goal doc
    ├── pipeline/data-pipeline.md          # pfsrd2 pipeline: web → parser → data → S3 → API
    ├── infrastructure/aws-infrastructure.md  # Three-layer Terraform stack (run with tofu), accounts, CI/CD
    └── standards/engineering-standards.md    # Cross-repo and per-repo conventions
```

## Architecture Documents in Other Repos

Individual repos maintain their own detailed documentation. This repo references these and does not duplicate them.

### lets-roll
- `docs/impersonation.spec.md` — Impersonation feature specification
- `docs/pixel.spec.md` — Pixels digital dice integration spec
- `docs/impersonation_monitoring.md` — Monitoring infrastructure for impersonation

### pfsrd2-parser
- `docs/architecture.md` — Multi-stage parsing pipeline architecture
- `docs/architecture-critique.md` — Critical assessment of parser design
- `docs/schema-guide.md` — JSON schema guide
- `docs/enrichment-process.md` — Enrichment pipeline documentation
- `docs/ability-enrichment.md` — Ability data enrichment
- `docs/monster-families.md` — Monster family structures
- `docs/monster-templates.md` — Monster template system
- `docs/adding-new-parser.md` — Guide for new parser modules
- `docs/aon-html5-error-patterns.md` — Common AoN HTML issues
- `docs/equipment-bucket-refactoring-plan.md` — Equipment refactoring plan
- `docs/cardinality-audit-process.md` — Data relationship auditing
- `docs/unified-ability-parser-status.md` — Unified ability parser status

### 521-engine
- `docs/design/00-index.md` — Design document index
- `docs/design/01-architecture.md` — Architecture overview
- `docs/design/02-game-objects.md` — Game object format specification
- `docs/design/03-dag-resolver.md` — DAG resolver algorithm
- `docs/design/04-engine-instance.md` — Per-entity runtime API
- `docs/design/05-session.md` — Multi-instance session layer
- `docs/design/06-hydration-serialization.md` — JSON round-trip and standalone export
- `docs/design/07-pf2e-integration.md` — PF2e integration strategy
- `docs/design/08-creature-schema-migration.md` — Flat stat block → DAG migration
- `docs/design/09-transparency-and-gm-authority.md` — explain() UX and GM overrides
- `docs/design/10-design-philosophy.md` — Opt-in complexity philosophy
- `docs/design/11-open-questions.md` — Open architectural questions
- `docs/design/12-roadmap.md` — Implementation roadmap

## The 521 Studios Ecosystem

### Product Philosophy
**Opt-in complexity.** Focused tools that compose, not a monolithic VTT. Start simple, add depth when you want it.

### Service Catalog
See `docs/architecture/system-overview.md` for the full service catalog with ownership.

### Data Pipeline
See `docs/pipeline/data-pipeline.md` for the pfsrd2 HTML → JSON → S3 → API pipeline.

### Infrastructure
See `docs/infrastructure/aws-infrastructure.md` for the three-layer Terraform architecture.

### Deployment Order
```
infra (baseline) → app terraform (lets-roll, pfsrd2-data-api) → infra-frontend (CloudFront/DNS)
```
Each layer reads the previous via `terraform_remote_state`. Deploy in order.

## Agent Fleet

The live roster and team rules are in `~/521Studios/TEAM.md`, maintained by
`pm-521-studios`. Read it at session start. Status and questions go to the PM;
workers may message each other directly. The steward table is in
`docs/architecture/system-overview.md`.
