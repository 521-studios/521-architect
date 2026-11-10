# External Architecture Documentation Index

Architecture knowledge lives in individual repos alongside the code it describes. This index maps what exists where, so Abadar (and anyone else) can find the authoritative source for any topic.

## lets-roll

| Doc | Path | Covers |
|-----|------|--------|
| Impersonation spec | `docs/impersonation.spec.md` | Support staff impersonation feature: scope, permissions, auto-expiry, audit trail |
| Pixel spec | `docs/pixel.spec.md` | Systemic Games Pixels digital dice: Bluetooth pairing, roll integration, advantage/disadvantage |
| Impersonation monitoring | `docs/impersonation_monitoring.md` | Monitoring and testing infrastructure for impersonation |

**Also:** `CLAUDE.md` is comprehensive — covers full app architecture, DB schema, deploy pipeline, dev setup.

## pfsrd2-parser

The most documentation-rich repo. Covers both architecture and operational guides.

| Doc | Path | Covers |
|-----|------|--------|
| Architecture | `docs/architecture.md` | Multi-stage pipeline: passes, layers, error handling philosophy |
| Architecture critique | `docs/architecture-critique.md` | Honest assessment of parser design — where it's strong, where it's fragile |
| Schema guide | `docs/schema-guide.md` | JSON schema definitions and versioning strategy |
| Enrichment process | `docs/enrichment-process.md` | Post-parse enrichment pipeline (SQLite DBs + Qwen LLM) |
| Ability enrichment | `docs/ability-enrichment.md` | Structured ability data enrichment |
| Monster families | `docs/monster-families.md` | How creature family groupings are structured |
| Monster templates | `docs/monster-templates.md` | Template system (Elite, Weak, etc.) at the parser level |
| Adding new parser | `docs/adding-new-parser.md` | Step-by-step guide for new content type parsers |
| AoN HTML errors | `docs/aon-html5-error-patterns.md` | Catalog of common HTML issues encountered in AoN source |
| Equipment refactoring | `docs/equipment-bucket-refactoring-plan.md` | Plan for reorganizing equipment data extraction |
| Cardinality audit | `docs/cardinality-audit-process.md` | Process for auditing data field relationships |
| Unified ability parser | `docs/unified-ability-parser-status.md` | Status of the unified ability parser implementation |

## 521-engine

Comprehensive design specification in 13 documents. This is the most architecturally detailed repo — the engine is in design phase and the docs are the primary artifact.

| Doc | Path | Covers |
|-----|------|--------|
| Index | `docs/design/00-index.md` | Overview and reading order |
| Architecture | `docs/design/01-architecture.md` | Layers, principles, host model |
| Game objects | `docs/design/02-game-objects.md` | Universal JSON+JS format for all game content |
| DAG resolver | `docs/design/03-dag-resolver.md` | Resolution algorithm, topological sort, stacking rules |
| Engine instance | `docs/design/04-engine-instance.md` | Per-entity runtime, API, explain() debugging |
| Session | `docs/design/05-session.md` | Multi-instance container, cross-instance effects |
| Hydration | `docs/design/06-hydration-serialization.md` | JSON round-trip, function compilation, standalone export |
| PF2e integration | `docs/design/07-pf2e-integration.md` | Strategy for integrating PF2e rules as game objects |
| Creature migration | `docs/design/08-creature-schema-migration.md` | Migrating flat stat blocks to assembled DAG nodes |
| Transparency | `docs/design/09-transparency-and-gm-authority.md` | explain() UX design, GM override mechanisms |
| Design philosophy | `docs/design/10-design-philosophy.md` | Opt-in complexity, progressive enhancement |
| Open questions | `docs/design/11-open-questions.md` | Unresolved: cross-instance deps, performance, damage formulas |
| Roadmap | `docs/design/12-roadmap.md` | Phased implementation plan |

## pfsrd2-data-api

| Doc | Path | Covers |
|-----|------|--------|
| CLAUDE.md | `CLAUDE.md` | Full API architecture, endpoints, template engine, deployment |
| Agent reviewers | `AGENT-REVIEWERS.md` | Automated review rules (test coverage, OpenAPI, error handling, SQL injection, complexity) |

## pfsrd2-display

| Doc | Path | Covers |
|-----|------|--------|
| CLAUDE.md | `CLAUDE.md` | Component architecture, rendering pipeline, dev harness, known issues |
| Agent reviewers | `AGENT-REVIEWERS.md` | Automated review guidelines |

## tools

| Doc | Path | Covers |
|-----|------|--------|
| Prerequisites | `docs/prerequisites.md` | AWS CLI, SSO, Session Manager Plugin setup |
| Usage | `docs/usage.md` | Full reference for db, console, logs tools |

## documentation

| Doc | Path | Covers |
|-----|------|--------|
| AWS access setup | `aws-access-setup.md` | SSO configuration, profile naming, MFA, daily workflow |

## What's Missing

Areas without dedicated documentation (potential Abadar action items):
- **pfsrd2-automation** — No architecture docs. Build/upload process documented only in CI config and CLAUDE.md.
- **pfsrd2-data** — No architecture docs beyond CLAUDE.md. Schema versioning strategy is oral knowledge.
- **pfsrd2-web** — No architecture docs. The download/normalize/patch workflow is documented only in shell script comments and CLAUDE.md.
- **Cross-repo data contracts** — No single doc describing the game_id system, schema version compatibility, or S3 bucket structure as a contract between repos.
- **ADRs (Architecture Decision Records)** — No formal decision log anywhere. Decisions live in commit messages, PR descriptions, and conversations.
