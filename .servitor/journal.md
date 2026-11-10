# Abadar — Journal

## 2026-04-01 — Initialization

First Vault established. Initial architecture documentation written covering:
- System overview and service catalog
- Data pipeline architecture (pfsrd2-web → parser → data → automation → S3 → API)
- AWS infrastructure (three-layer Terraform stack)
- Service dependency map (runtime + build-time)
- Service extraction roadmap and principles
- Cross-repo engineering standards

Cataloged existing architecture docs across all repos (pfsrd2-parser: 12 docs, 521-engine: 13 design docs, lets-roll: 3 feature specs, tools: 2 docs).

Standing orders: actively listen for decisions across fleet communications and record them.

## 2026-08-14 — Encounter Builder reconciliation

Reconciled the architectural record against shipped state (docs had drifted since
init; goal doc still listed decisions as open). Ground-truthed via beads.

- **Goal `encounter-builder-usable`**: workstreams A (OIDC provider `z3qj`) and B
  (SPA `7v94`, api `1p1a`, X-Access-Token `09ae`, chapters `be75`, encounter-v2
  `sdmq`, e2e harness `r2ex`) shipped and staging-validated. Added a status banner,
  resolved the three "decisions I need from you," updated "done when."
- **Design `encounter-builder-v2`**: epic `0ybu` shipped; marked SHIPPED banner,
  resolved open decisions (chapters first-class, elite/weak-first templates,
  budgeting shipped as epic `ncnq`).
- **Remaining live item**: workstream C — prod cut-over (bead `pmib`, OPEN),
  gated on Devon's explicit prod authorization. Recorded in state.json.
