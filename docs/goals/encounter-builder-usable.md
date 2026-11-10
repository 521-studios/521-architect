# /goal — Encounter Builder usable end-to-end (Phase 1 close-out)

## Status — 2026-08-14: staging done, prod pending

**Workstreams A + B shipped and staging-validated.** The GM side is real:
OIDC login, campaign list, encounter CRUD, monster/treasure/coin editors,
masking, `CreatureStatBlock` render, persistence, and release all work on
`encounters.staging.521studios.com`, every piece merged through the review loop.
Beads closed: OIDC provider `z3qj` (workstream A), SPA `7v94`, api `1p1a`,
X-Access-Token `09ae`, chapters `be75`, encounter-v2 `sdmq`, Playwright e2e
harness `r2ex`, plus the v2 epic `0ybu` and the party-treasure budgeting epic
`ncnq` (which overran into the follow-on design below).

**Remaining live item: workstream C — production cut-over (bead `pmib`, OPEN).**
Deferred to explicit prod authorization from Devon. That is the only thing
between here and the goal's "done when."

The follow-on two-pane / chapters / templates work is designed in
[`../design/encounter-builder-v2.md`](../design/encounter-builder-v2.md) and
also shipped (epic `0ybu`).

## North star (definition of done)

A GM opens **https://encounters.staging.521studios.com**, logs in with their
lets-roll account, picks one of their campaigns, builds an encounter (monsters +
treasure + coin), masks the magic items, and **releases** it — every action
persisted through the live `encounter-builder-api`. Monster stat blocks render
via `pfsrd2-display`. Then the same flow works in **production**.

Party Treasure (the app that *receives* a release) is the **next** goal, not this
one. This goal ends when the GM side is real and shippable.

## Acceptance walkthrough (this is the test)

1. Hit `encounters.staging.521studios.com` unauthenticated → redirected into
   lets-roll's OIDC login (Authorization Code + PKCE) → back to the SPA logged in.
2. SPA shows the GM's campaigns (from `GET /api/v1/games`, GM ones flagged).
3. Create an encounter in a campaign; add monsters by `game_id` (pfsrd2 lookups via
   `/api/pfsrd2/*`), set elite/weak; add treasure lines (pristine/derived/custom
   refs), mark some magic items masked with an identify DC; set coin.
4. Stat blocks render with `CreatureStatBlock`; elite/weak adjustments visible.
5. Save → reload → state persists (through `encounter-builder-api` → DynamoDB).
6. Release the encounter → status flips to `released`, becomes read-only (409 on
   edit). (The handshake to Party Treasure is stubbed/deferred to the next goal.)
7. All API calls carry the OIDC bearer in **`X-Access-Token`** (CloudFront OAC
   eats `Authorization`); non-GM/non-member is 403, unauth is 401.
8. Repeat 1–6 against **production** at `encounters.521studios.com`.

## Workstreams

### A. lets-roll OAuth app registration  *(blocks login — needs a decision)*
- Register a first-party OIDC client `encounter-builder-web` in lets-roll
  (Doorkeeper) with `skip_authorization`, PKCE, redirect URIs
  `https://encounters.staging.521studios.com/auth/callback` and the prod host.
- Confirm the SPA's `OIDC_AUDIENCE`/client_id matches the API's `OIDC_AUDIENCE`
  (currently defaulted to `encounter-builder-web`).
- **Decision needed:** do this as a lets-roll seed/migration (reviewable, durable)
  vs. a console/console-script one-off. Recommend a seed so staging+prod match.

### B. encounter-builder-web React SPA  *(bd_521Studios-7v94 — the main build)*
Builds into the S3 bucket + CD pipeline that are already live. Suggested slices,
each its own PR + review loop, deploying to staging as they land:
1. **Vite + React shell + OIDC PKCE**: login/callback/logout, token store + silent
   refresh, an API client that attaches the bearer as `X-Access-Token`. Replaces
   the placeholder. Acceptance: login round-trip + `/api/app/me` renders the user.
2. **Campaign list**: `GET /api/v1/games`, GM filter, pick a campaign.
3. **Encounter CRUD UI**: list/create/edit/delete against `/api/app/campaigns/:id/
   encounters`; monster + treasure + coin editors; masking + identify DC; contentRef
   passed opaquely.
4. **pfsrd2-display integration**: consume `@521studios/pfsrd2-display` (git dep),
   render `CreatureStatBlock`; monster search via `/api/pfsrd2/*`.
5. **Release**: the release action + released/read-only states.
- Carry the frontend `AGENT-REVIEWERS.md` pack now in the repo.

### C. Production cut-over  *(deferred — needs explicit prod authorization)*
- Prod deploy roles + `AWS_ROLE_ARN_PROD` for `encounter-builder-api` and
  `encounter-builder-web` (prod account **242258465702**), each trusting the
  **ID-augmented** OIDC subject `repo:521-studios@216783907/<repo>@<repo_id>:*`.
- Prod wiring of `encounter-builder-cf` in `infra-frontend/terraform/environments/
  production` (mirrors staging; `encounters.521studios.com`).
- First prod applies of api + web + infra-frontend (manual dispatch).
- **Fleet follow-up (separate):** `bootstrap.sh` emits the *plain* OIDC subject;
  fix it and backfill the existing repos' trust policies. See
  memory `project_github_oidc_subject_ids`.

## Constraints carried forward (don't relitigate)
- **Bearer travels in `X-Access-Token`** behind CloudFront (OAC overwrites
  `Authorization`). SPA must send it there; API already reads it.
- **Any new deploy role** must trust the ID-augmented OIDC subject, not the plain
  `repo:org/repo:*` form.
- **Staging = continuous delivery** (auto-apply on merge); **prod = manual dispatch**.
- **All work via PR** + agent review loop (bd jsonl commits exempt). Staging AWS
  changes OK autonomously; **prod never without explicit authorization**.
- contentRefs are stored/passed **opaquely**; rendering/derivation lives in
  pfsrd2-data-api / pfsrd2-display, never in encounter-builder-api.
- Encounter Builder is **GM-only**; players never touch it (that's Party Treasure).

## Decisions (resolved 2026-08-14)
1. **OAuth registration** — resolved: shipped as the lets-roll OIDC provider
   (Doorkeeper + PKCE + JWKS, bead `z3qj`), the durable/reviewable path.
2. **SPA scope for "done"** — resolved: the acceptance walkthrough was the bar for
   this goal; treasure budgeting / trades landed in the follow-on `ncnq` cluster.
3. **When to cut prod** — resolved: after staging passes (it has). Prod cut-over is
   held as bead `pmib`, awaiting Devon's explicit prod authorization.

## Done when
The acceptance walkthrough passes on **staging and production**, every piece
merged through review, and `7v94` (+ this goal's OAuth and prod tickets) are
closed. **Status: staging met, all beads closed except `pmib` (prod cut-over).**
Next goal: Party Treasure + the release handshake.
