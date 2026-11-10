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

## 2026-11-10 — Team intro, catalog, ContentRef contract

Introduced to the team (roster: ~/521Studios/TEAM.md, PM pm-521-studios).
Devon's decisions: lets-roll owns live combat (Encounter Builder prep-only);
engine content is declarative for GMs, vetted compute built ahead of time with
no runtime eval; stacking rules are data; 4zx0 waits for engine Phase 5.

- Wrote `docs/architecture/content-ref.md` (DRAFT): closed wrapper, open
  `{kind,id,params,v}` elements, modifications = truth / json = cache, pinned
  versions never pruned, content-hashed compute ids.
- Catalog: added encounter-builder, party-treasure, wyrd, lets-roll OIDC;
  steward table; pfsrd2 parser/data/web/corpus under pm-masterwork-tools.
- Reviewed muz0 map images (approved with pending-tag expiry, presigned POST,
  HeadObject on save). party-treasure-web masked items → ItemCard (library fix).
- Open for Devon: wyrd standalone Terraform intentional?; release push went
  server-to-server vs the cluster plan's browser handoff — record why.
- Devon: release push server-to-server is deliberate (backend op is easier to
  secure and test); wyrd terraform folds into infra later (t65j). New stewards:
  521-infra, 521-infra-frontend.
