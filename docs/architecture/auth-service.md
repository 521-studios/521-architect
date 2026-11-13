# Auth Service and the Retirement of the lets-roll Backend

Status: PROPOSED (2026-11-13). Direction: Devon. Bead: `bd_521Studios-qxlz`.
It grew out of the `we5k.2.6` lets-roll↔EB auth discussion.

## Direction (Devon, 2026-11-13)

- "a microservice that is permissions/JWT for all apps to use, which would mean porting lets-roll to that."
- "eventually, we should move the users and permissions into dynamo, but I'm ok with phasing this."
- "I'm ok with lets roll having users in it's own data store, just we have a principal that would be in dynamo, and if that user uses lets roll, it can create a user record."
- "long term I want to get rid of the rails backend entirely, it's just a matter of migrating piece by piece."

## Model

**The auth service owns principals.** It is a Lambda with DynamoDB, like EB and PT.
A principal is an identity: its credentials, its email, and service clients (lets-roll↔EB, and the M2 agents).
The auth service issues every token. Apps verify tokens locally against its JWKS.

**Apps own their users.** Each app creates its own user record the first time a principal uses it, keyed by `sub`.
lets-roll keeps its `users` table and its foreign keys, and gains a `principal_id` column. Only sign-in changes: Devise passwords move to the auth service.

**Apps own their permissions.** The app that owns a resource decides who may act on it.
For example, lets-roll decides who is GM of a game, and PT decides who can see a masked item.
Principals carry only global roles, such as `admin`.

**One rule for cross-app checks: ask the owner.** An app that needs another app's permission (for example, "is this user GM of game 12?") calls the owner's API and caches the answer briefly. Today's `GET /api/v1/games/:id` already works this way.
Permissions are not put into token claims. A claim would go stale until the next refresh, and a GM of many games would carry a large token.

**`sub` stays stable.** EB and PT store owners by today's `sub`, the lets-roll user id.
- Principals are seeded with the existing ids as their `sub`.
- New principals get ULIDs.
- Every app treats `sub` as an opaque string.

## Build or adopt

Devon's standing rule: check maintenance and licence before recommending software. Checked 2026-11-13.

| Option | Maintained / licence | Fit |
|---|---|---|
| **Own service on a maintained library** (Go Lambda + DynamoDB, protocol from [`zitadel/oidc`](https://pkg.go.dev/github.com/zitadel/oidc/v3)) | The library is Apache-2.0 and actively released. | We control `sub`, claims and storage, and it matches the Lambda + Dynamo pattern. The cost is a KMS signing key, about $1/mo per env. We own the security-sensitive code, but not the protocol. |
| **Amazon Cognito** | Managed by AWS. | Serverless, and $0 at our size: 10k MAU are free on the Lite and Essentials tiers. Machine-to-machine tokens cost $0.00225 per token request, which is negligible with caching. A user-migration trigger can move bcrypt users lazily. **But `sub` can't be set, added, overridden or suppressed**, so EB and PT would have to rekey their owners or read a custom claim. It is also an AWS lock-in, and only lightly customisable. |
| **Keycloak** | Active (26.8.2, Nov 2026). Apache-2.0. | An always-on JVM server plus Postgres. That cuts against the Lambda + Dynamo direction and the rule against always-on cost. It's far more than a dozen users need. |
| **Ory Kratos + Hydra** | Apache-2.0 open source, but some features, and security patches between open source releases, ship only under the Enterprise License. | Two always-on servers plus Postgres, and patch latency on the open source tier. |

**Recommendation: build our own service on `zitadel/oidc`.**
- Use the library for OIDC and OAuth2: authorization code with PKCE, client credentials, refresh tokens, and the device grant for M2 agents. Don't hand-roll the protocol.
- Sign tokens with a KMS asymmetric key, so the private key never sits in the Lambda.
- Cognito is the credible alternative if Devon would rather own less security code. Its price is the `sub` rekey.

## Phases

**Phase 0: lets-roll's Doorkeeper is the issuer (now).** lets-roll is already the OIDC provider. Its access tokens are 15-minute RS256 JWTs, and it publishes a JWKS.
- Add the `client_credentials` grant to Doorkeeper for service clients. Today `grant_flows` is `authorization_code` only.
- Make every app read its issuer and JWKS URL from config.
- Moving to the auth service later is then a config change in every app.

**Phase 1: stand up the auth service, then move principals into Dynamo.**
- The auth service starts issuing tokens with principals in Dynamo. Seed it with the lets-roll user ids, emails and bcrypt hashes, and dual-write from lets-roll only briefly.
- lets-roll's sign-in moves to the auth service, and lets-roll becomes a relying party.
- Flip each app's issuer config.

This deliberately skips the step where the auth service reads lets-roll's Postgres directly, which Devon was OK with.
That step would put the auth Lambda in the VPC, which needs a NAT gateway or interface endpoints for KMS and STS, against the "no new NAT unless earned" rule. It would also share a database across services, which the extraction roadmap lists as an anti-pattern.
Phase 0 already gives every app a working issuer, so nothing needs to read Postgres from outside lets-roll.

**Phase 2: retire the Rails backend in strangler style.** Each piece becomes a Lambda + Dynamo service, in the order below.

### DynamoDB notes

- **Unique email.** Use one `TransactWriteItems`: put `PRINCIPAL#<sub>` and put `EMAIL#<normalised email>`, each with `attribute_not_exists`.
  - An email change deletes the old claim item and puts the new one in the same transaction.
  - Normalise by lowercasing and trimming. Don't strip dots or `+` tags.
- **TTL.** Refresh tokens, revocations and invites carry a TTL.
  - TTL deletion is lazy and can take days, so every read compares the expiry itself and never relies on the item being gone.

## Near-term: lets-roll↔EB service auth (`we5k.2.6`, `lets-roll-f6e`)

This revises 521-architect's earlier IAM pick for lets-roll→EB.
That pick was right for that one call. But Devon's direction makes the issuer's service tokens the standard for every service call, and IAM would be a second mechanism we'd later remove.

- **Service calls use client-credentials JWTs from the configured issuer.** That's Doorkeeper in phase 0 and the auth service in phase 1.
  - Each service has its own client.
  - Tokens are scoped per route (`eb:kill-report`, `eb:pull`, `lr:eb-push`) and have a short `exp` (5 minutes).
  - The caller caches its token until close to expiry.
- **lets-roll → EB (kill reports, the `.2.2` pull).**
  - EB verifies locally, with the same verifier it uses for GM tokens.
  - The token goes in `X-Access-Token`, because CloudFront OAC owns `Authorization`.
  - EB tells service tokens apart from GM tokens by `aud` and scope, and each route accepts only its own kind.
- **EB → lets-roll (the push).** The push carries EB's service token and the GM's bearer. lets-roll verifies both locally, then checks GM of the game.
- **What this fixes.** A forged token fails a local signature check, with no outbound call, so the amplification in `we5k.2.6` is gone.
  - The verifier must rate-limit JWKS refetches on an unknown `kid` (at most once a minute), or the amplification moves there.
  - Each forgery still costs one Lambda invocation. Reserved concurrency can cap that if needed.
- **What it gives up.** A bearer token is not bound to the method, path or body, as the `#12` binding was.
  - Per-route scope plus the 5-minute `exp` limits a stolen token to one kind of call for a few minutes.
  - DPoP can add that binding later if ever needed.
- **Effect on lets-roll's chain.**
  - `jlm`, `6ru` and `dur` are unchanged.
  - `f6e` becomes a JWT check instead of an STS proof, which is less code.
  - The lets-roll→EB client becomes "fetch a client-credentials token, send it", replacing the STS presign.
  - EB deletes `internal/lrident`.
- **Timing.** Kill sync does not need to stay dark until the auth service exists. It can go live once Doorkeeper issues client credentials, which is phase 0.

## Availability

- An auth outage blocks sign-in and refresh. It does not block verification of tokens already issued.
- Verifiers cache the JWKS and keep serving the cached copy when a fetch fails.
- User access tokens are 15 minutes, so a session lasts at most 15 minutes past the start of an outage.
  - If that's too short for a game night, raise the access TTL.
  - Don't make verification depend on the auth service.
- Key rotation: publish the new key in the JWKS at least one cache lifetime before signing with it, and keep the old key until its tokens expire.

## Where it sits

**A new milestone, M5, after M4.**
- M4 (prod cut-over before the next campaign) runs on lets-roll's existing provider and doesn't need the auth service.
- Phase 0 is small and belongs to `we5k.2` under M3. M2's agent auth can use phase 0 (the device grant) and move with the issuer.
- Under the freeze (one heavy line on ares), phases 1 and 2 are their own line. They start after the current line (`muz0`, then M1) and M4 finish.

## Rails retirement order

Pieces leave in order of how many apps consume them (the extraction rule) and how risky they are.

1. **Auth and principals** (this doc). Impersonation moves with it, as token exchange in the auth service.
2. **Games and membership** (`gonf`). EB and PT both read them (`fcu0.2`), so there are several consumers.
3. **Quantum dice.** It's standalone, and a design exists (`~/521Studios/quantum-dice-fairness-system.md`).
4. **Characters, shortcuts and variables.**
5. **Rolls, initiative, live combat and realtime** (ActionCable, Pixels). These are the core and the largest piece; they need a websocket plan (for example, API Gateway WebSockets). They go last.

When the last piece leaves, lets-roll becomes a static SPA over these services.

## Decisions for Devon

1. Build on `zitadel/oidc` (recommended), or adopt Cognito and accept the `sub` rekey?
2. Phase 0 instead of reading lets-roll's Postgres from the auth service?
3. Client-credentials JWTs for `we5k.2.6` (recommended), instead of the IAM pick?
4. M5 after M4?
