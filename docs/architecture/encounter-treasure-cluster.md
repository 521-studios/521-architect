# Encounter Builder + Party Treasure — Cluster Plan

Status: PROPOSED (2026-08-11). Planner: Claude (Fable). Owner: Devon.

Two new apps joining the 521 Studios cluster: **Encounter Builder** (GM builds
encounters with monsters + treasure for a campaign) and **Party Treasure**
(players receive, identify, trade, split, and sell released treasure). Both are
discrete apps — usable alone, better together — sharing AAA through lets-roll.

This plan follows the standing ecosystem rules from `system-overview.md` /
`service-dependencies.md`: opt-in complexity, shared-nothing services, **the
browser is the integration point**, cost-disciplined edge (reuse distributions
where sensible, path-based routing, no new NAT/ALB unless earned).

---

## 1. Repo map

New repos (all in 521Studios workspace, all with their own terraform per the
app-repo pattern):

| Repo | What | Stack |
|---|---|---|
| `encounter-builder-api` | Backend for encounters/treasure-prep | Go 1.24, chi, Lambda (arm64, Function URL + IAM), DynamoDB |
| `encounter-builder-web` | GM-facing SPA | React 18 + Vite, consumes pfsrd2-display |
| `party-treasure-api` | Backend for hoard/ledger/trades/sales | Go 1.24, chi, Lambda, DynamoDB |
| `party-treasure-web` | Player+GM SPA | React 18 + Vite |

Modified repos:

| Repo | Change |
|---|---|
| `lets-roll` | Becomes the OIDC identity provider (Doorkeeper); gains a small `/api/v1` for campaign/membership reads; rack-cors |
| `infra-frontend` | Two new CF modules cloned from `wyrd-cf` + DNS records (after committing the currently-untracked `wyrd-cf`/`lets-roll-cf` work on `ci/gate-auto-apply`) |
| `pfsrd2-display` | Made consumable as a git dependency; later: item-card component |
| `smoke_tests` | Playwright coverage for login → build → release → distribute |

`infra` (the fabric) needs **no changes** — no VPC resources required
(DynamoDB + Lambda are VPC-free), consistent with pfsrd2-data-api.

## 2. AAA — lets-roll as identity provider

lets-roll owns users (Devise, cookie sessions, no token support today). Rather
than extracting an auth service (fails the extraction checklist — no stable
contract exists yet), lets-roll **becomes the OIDC provider** for the cluster:

- **Gems**: `doorkeeper` + `doorkeeper-openid_connect`. Authorization Code +
  PKCE for the SPAs (public clients, no client secret in the browser). Signed
  JWT access tokens (RS256) + `/.well-known/jwks.json` + discovery doc.
- **Login UX**: SPA redirects to lets-roll's existing Devise sign-in, user
  approves once, redirected back with a code, exchanged for tokens in the
  browser. Existing lets-roll sessions mean returning users click through
  without re-entering credentials — effective SSO.
- **Verification in Go APIs**: one small JWT middleware per API — fetch JWKS
  (cache aggressively, keys rotate rarely), verify `iss`/`aud`/`exp`, extract
  `sub` (user id). Stdlib + `lestrrat-go/jwx` or equivalent; no calls to
  lets-roll on the request path.
- **Authorization** stays out of the token. GM-ness and party membership are
  per-game facts owned by lets-roll; sibling APIs fetch them from `/api/v1`
  (below) with the caller's bearer token and cache briefly (60s). Tokens stay
  dumb: identity only.
- **Accounting**: token grants are Doorkeeper rows (auditable); API access
  logs are CloudWatch per Lambda. No new accounting infra.
- **CORS**: `rack-cors` on lets-roll, allowlisting
  `https://*.521studios.com` + `https://*.staging.521studios.com` origins for
  the `/oauth/*` + `/api/v1/*` paths only. Bearer-token calls, no cookies →
  no credentialed CORS. The rest of lets-roll stays same-origin.

Explicitly **not** doing: shared cookie domain / shared `SECRET_KEY_BASE`
(couples every app to Rails cookie internals, and Go Lambdas would have to
reimplement Rails cookie decryption), a standalone auth service (premature
extraction), or touching the legacy MD5 `private_token`s (Pixels/viewer paths
keep working unchanged).

## 3. lets-roll `/api/v1` — campaign & membership reads

Small, token-authed (Doorkeeper), JSON-only namespace. This is the "let other
apps AAA via lets-roll's db" surface:

```
GET /api/v1/me                      → { id, username, email }
GET /api/v1/games                   → games where I'm GM or have a character
GET /api/v1/games/:id               → { id, name, gm_user_id, am_gm }
GET /api/v1/games/:id/members       → [ { user_id, username, character_id, character_name } ]
```

Read-only in v1. GM = `games.user_id` (the existing convention). No
Campaign/Party model work in lets-roll — `Game` *is* the campaign, membership
*is* having a character, and the sibling apps adopt those semantics as-is.

## 4. Data layer for the new APIs

**DynamoDB, single table per service** (pattern precedent: pfsrd2-data-api's
`defects` table). Serverless, zero VPC, on-demand billing ≈ $0 at this
traffic. Aurora would drag both Lambdas into the VPC and add connection
management for no modeling win — these are document-shaped aggregates with
key-value access patterns.

### Content references: the ref | inline convention

Monsters and items in the dataset are referenced by `game_id` and rendered
from pfsrd2-data-api at view time — same rule lets-roll's
`encounter_monsters` already follows. But content **mutates**: pfsrd2-data-api
already applies templates to monsters and accepts *inline creature JSON*
(`POST /templates/apply`), and items get constructed (a 3rd-level scroll +
fireball = a fireball scroll; a dagger + +2 rune + striking rune = a +2
striking dagger). The longer-term direction is builders that start from an
existing entry and alter it.

So every field in this cluster that holds "a monster" or "an item" uses one
shape, everywhere:

```
contentRef :=
  { game_id, version? }                          # pristine dataset entry
| { base: {game_id, version?},                   # derived: provenance kept
    modifications: [...],                        #   templates, runes, spell,
    json: {...} }                                #   + the resolved snapshot
| { json: {...} }                                # fully custom / hand-built
```

Rules: (a) APIs and display components always accept the resolved `json` the
same way they accept fetched-by-game_id data — pfsrd2-display is already
data-as-props, so mutated JSON renders for free; ItemCard must follow suit.
(b) Derived objects keep `base` + `modifications` as provenance, not just the
snapshot, so a future builder can re-open and re-edit rather than treating
the result as opaque. (c) Construction/derivation logic (item assembly,
template application) belongs in **pfsrd2-data-api** (or 521-engine behind
it) mirroring the template-apply endpoints — the encounter/treasure apps
store and pass contentRefs, they don't implement game rules. (d) DynamoDB's
400KB item ceiling is fine for item JSON and nearly all creatures; if a
snapshot ever exceeds it, spill the `json` to S3 and keep a pointer — noted,
not built.

The builders themselves are **not designed here**. The monster editor will be
its own app, **Monster Lab** (already in 521-architect's planned-services
catalog), and the item/treasure builder its sibling, **Treasure Lab** — same
start-from-existing-and-alter pattern (pick a scroll, attach fireball; pick a
dagger, slot runes). This convention is the accommodation so both can arrive
without schema migrations: anything either Lab produces is just another
contentRef (`base` + `modifications` + resolved `json`) that Encounter
Builder and Party Treasure already know how to store, pass, and render.

### encounter-builder-api model

```
PK=CAMPAIGN#<letsroll_game_id>            # campaign settings: party size/level, name cache
PK=CAMPAIGN#<id> SK=ENCOUNTER#<ulid>      # name, status(draft|run|released), notes
  monsters: [ { ref: contentRef, count, adjustment: none|elite|weak, nickname? } ]
  treasure: [ { line_id, ref: contentRef,
               qty, masked: bool, mask_label ("unidentified potion"),
               identify_dc (default = f(level, rarity), GM-overridable),
               sale_class: normal|pure_treasure,
               state: intact|consumed|destroyed, state_note ("drunk by ogre") } ]
  currency: { cp, sp, gp, pp }
  released_at, released_to (treasure party id) — set after release
```

XP budgeting (trivial→extreme thresholds, creature XP from level delta,
elite/weak ±1 level) is a **pure client-side table** — PF2e XP is a function
of party level vs creature level; there is no XP in the dataset and none is
needed.

### party-treasure-api model

```
PK=PARTY#<letsroll_game_id>               # party settings: default sale percent (GM),
                                          # per-category overrides optional later
PK=PARTY#<id> SK=ITEM#<ulid>              # one physical item (qty split on receipt)
  ref: contentRef                          # pristine | derived | custom (see §4)
  masked: bool, mask_label, identify_dc
  identified_at?, identified_by?
  owner: POOL | CHARACTER#<letsroll_character_id>
  for_sale: bool, sale_percent (item override; pure_treasure ⇒ 100)
  status: held|sold|removed
  history: [ { at, actor, event } ]        # append-only per-item audit trail
PK=PARTY#<id> SK=LEDGER#<ulid>            # currency events (append-only)
  kind: release|sale|split|assign|adjust
  amounts {cp,sp,gp,pp}, allocations: [ { character_id|POOL, share } ]
PK=PARTY#<id> SK=TRADE#<ulid>             # offer/accept between two characters
```

Money splitting supports **direct assignment** and **shares** (N shares,
party-fund share optional — the classic "party gets a share" convention);
split math is client-side, the ledger event records the resolved allocation.
Sales credit `price × sale_percent` (GM-set default, per-item override,
pure-treasure fixed at 100%) to the pool as a ledger event. Item prices come
from the pfsrd2 full JSON (`stat_block.price.value` is structured); the
indexed `price` string is display-only — don't parse it.

**Masking**: players see `mask_label` only until identified. Enforced
server-side — the API strips `ref.game_id` and real names from responses to
non-GM callers while `masked && !identified`. Identify flow: player rolls (in
lets-roll or at the table), GM or the roll outcome flips it; v1 keeps it
manual (GM clicks identify / player enters roll vs DC) — no lets-roll dice
integration yet (opt-in complexity; wire to lets-roll rolls later if wanted).

## 5. The release handshake (browser as integration point)

No server-to-server calls, per the standing rule:

1. GM finishes an encounter in Encounter Builder, prunes the treasure list
   (marks the potion `consumed`, the scrolls `destroyed` — filtered out or
   shown struck-through, GM's choice per line), confirms masks + identify DCs.
2. Encounter Builder (browser, GM's bearer token) `POST`s the release payload
   to `party-treasure-api /parties/:game_id/releases`.
3. party-treasure-api verifies the JWT, confirms the caller is GM of that
   game via lets-roll `/api/v1/games/:id` (cached), creates ITEM + LEDGER
   rows, returns a receipt.
4. Encounter Builder marks the encounter `released` with the receipt id.

Idempotency: release carries a client ULID; replays are no-ops. If the party
doesn't use Party Treasure, release is simply disabled/hidden — Encounter
Builder remains fully useful standalone (opt-in rule). Likewise Party
Treasure accepts manual GM entry, so it works without Encounter Builder.

## 6. Frontend + edge

Both SPAs clone the **wyrd pattern**: Vite build → S3 bucket owned by the app
repo's terraform → `wyrd-cf`-cloned CloudFront module in infra-frontend with
the SPA at bucket root and API origins mounted by path:

```
encounters.521studios.com            party-treasure.521studios.com
├── /            → S3 SPA            ├── /            → S3 SPA
├── /api/app/*   → encounter-builder ├── /api/app/*   → party-treasure-api
│                  -api Lambda (OAC) │                  Lambda (OAC)
├── /api/pfsrd2/* → pfsrd2-data-api  ├── /api/pfsrd2/* → pfsrd2-data-api
│                   Lambda (OAC)     │                   (item lookups)
└── (lets-roll /oauth + /api/v1 called cross-origin with CORS — no proxying)
```

Mounting pfsrd2-data-api as an origin on each distribution keeps it
same-origin (it has no CORS, deliberately) and reuses the exact OAC +
origin-request-policy recipe from `wyrd-cf` — including the known gotchas:
strip `Authorization`/`Host` from the forwarded headers or SigV4 breaks, POST
bodies need client-supplied `x-amz-content-sha256`, and SPA-rewrite must be a
CloudFront Function, not `custom_error_response`.

lets-roll endpoints are called cross-origin with bearer tokens + rack-cors —
proxying Rails through two more distributions buys nothing.

**Precondition**: infra-frontend's `ci/gate-auto-apply` branch has `wyrd-cf`
untracked and the old `cloudfront/` module deleted-but-uncommitted. That must
land via PR before cloning modules from it.

### pfsrd2-display consumption

Consume as a **git dependency** (`"@521studios/pfsrd2-display": "github:..."`
or workspace file path) — it already builds proper ESM/CJS bundles; no npm
publishing ceremony. Encounter Builder uses `CreatureStatBlock` for monster
display. Item cards don't exist in the library — build the item renderer *in*
pfsrd2-display (new `ItemCard` export) rather than in the app, since Party
Treasure needs it too. That's the second consumer that earns it a place in
the shared lib.

## 7. Phasing & work breakdown

Dependency spine: **AAA first, then APIs, then SPAs, edge last per app.**
Suggested model split: Opus for the design-heavy Go APIs and the Doorkeeper
integration; Sonnet for terraform clones, SPA scaffolding, and CI workflows.

**Phase 0 — Foundation (blocks everything)**
- lets-roll: Doorkeeper + doorkeeper-openid_connect (PKCE, RS256 JWT, JWKS,
  discovery), client registrations for both SPAs per env. [Opus]
- lets-roll: `/api/v1` read endpoints + rack-cors. [Opus, same PR series]
- infra-frontend: land the in-flight `wyrd-cf`/`lets-roll-cf` branch. [Devon/Sonnet]
- pfsrd2-display: git-dep consumability check (install from sibling app,
  verify dist artifacts committed or built on install). [Sonnet]

**Phase 1 — Encounter Builder**
- `encounter-builder-api`: repo scaffold from pfsrd2-data-api (chi, Lambda +
  local mode, terraform, CI), JWT middleware, DynamoDB table + CRUD,
  release-marking. [Opus]
- `encounter-builder-web`: repo scaffold from wyrd SPA layout, OIDC PKCE
  client, campaign picker (lets-roll `/api/v1`), encounter CRUD, monster
  search/suggest + `CreatureStatBlock`, XP budget widget, treasure list
  editor (item search, custom items, currency, masks + identify DCs).
  [Sonnet scaffold → Opus features]
- infra-frontend: `encounter-builder-cf` module + DNS both envs. [Sonnet]

**Phase 2 — Party Treasure**
- `party-treasure-api`: same scaffold; hoard/ledger/trades/sales endpoints,
  release intake with GM check + idempotency, masking enforcement
  server-side. [Opus]
- `party-treasure-web`: player pool/my-items views, identify flow, trades,
  split calculator (direct + shares), sale marking, GM settings (percents),
  GM manual entry. [Sonnet scaffold → Opus features]
- pfsrd2-display: `ItemCard` component. [Sonnet]
- infra-frontend: `party-treasure-cf` module + DNS. [Sonnet]

**Phase 3 — Integration & polish**
- Release flow end-to-end (Encounter Builder → Party Treasure).
- smoke_tests: Playwright journey (login via lets-roll → build encounter →
  release → distribute → sell).
- 521-architect: update service catalog + dependency map; this doc → ACCEPTED.

## 8. Risks / open questions

- **Doorkeeper × Devise fit**: well-trodden combo, but lets-roll's global
  `authenticate_user!` + CSRF need careful exemptions for `/oauth/token`.
  First Phase-0 PR should be a spike proving the full PKCE round-trip.
- **Game semantics**: "party" is pinned to a lets-roll game (membership =
  has-a-character). If ad-hoc parties without lets-roll games are ever
  wanted, that's a lets-roll feature, not a treasure-app feature.
- **Identify DC defaults**: computed from item level + rarity trait
  (rarity is a trait in the data, not a field). GM override always wins.
- **encounter_monsters overlap**: lets-roll already has run-time encounters
  (initiative/HP tracking). Encounter Builder is *prep*; lets-roll is *play*.
  A later "push encounter to lets-roll" integration is natural but out of
  scope — noted for the roadmap, not built now.
- **Token lifetime/refresh**: SPAs use short-lived access tokens + refresh
  token rotation (Doorkeeper supports it). Decide lifetimes in the Phase-0
  spike.
