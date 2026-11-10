# 521-architect

System architecture repository for 521 Studios. Maintained by the 521-architect session.

This repo contains the architectural record — service catalog, dependency maps, data pipeline documentation, infrastructure architecture, engineering standards, and extraction roadmap for the 521 Studios ecosystem.

No application code lives here. It is the authoritative reference for how everything connects.

## Quick Links

- [System Overview](docs/architecture/system-overview.md) — What we build, product philosophy, service catalog
- [ContentRef contract](docs/architecture/content-ref.md) — Cross-service shape for monsters/items and `modifications[]`
- [Service Dependencies](docs/architecture/service-dependencies.md) — Runtime and build-time dependency map
- [Data Pipeline](docs/pipeline/data-pipeline.md) — pfsrd2 HTML → JSON → S3 → API
- [AWS Infrastructure](docs/infrastructure/aws-infrastructure.md) — Three-layer Terraform stack
- [Extraction Roadmap](docs/architecture/extraction-roadmap.md) — Monolith decomposition plan
- [Engineering Standards](docs/standards/engineering-standards.md) — Cross-repo conventions
- [External Docs Index](docs/architecture/external-docs-index.md) — Architecture docs in other repos
