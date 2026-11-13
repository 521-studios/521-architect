# Auth Service and the Retirement of the lets-roll Backend

Status: ACCEPTED (Devon, 2026-11-13). Only the choice of builder is still open (see Who builds it). Bead: `bd_521Studios-qxlz`.
It grew out of the `we5k.2.6` lets-roll↔EB auth discussion.

## Direction (Devon, 2026-11-13)

- "a microservice that is permissions/JWT for all apps to use, which would mean porting lets-roll to that."
- "eventually, we should move the users and permissions into dynamo, but I'm ok with phasing this."
- "I'm ok with lets roll having users in it's own data store, just we have a principal that would be in dynamo, and if that user uses lets roll, it can create a user record."
- "long term I want to get rid of the rails backend entirely, it's just a matter of migrating piece by piece."
- On the plan: "build the auth service, deploy it, migrate everything, then we can cut over." Then: "fewer steps".
- On timing: "go for the right answer asap. even prod has zero users nearly every day."

Decisions: build on `zitadel/oidc`; no shared Postgres, with principals seeded into Dynamo; the auth service is built now, as the `we5k.2.6` fix; the Rails retirement is M5, after M4.

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

**Decision: build our own service on `zitadel/oidc`.**
- Use the library for OIDC and OAuth2: authorization code with PKCE, client credentials, refresh tokens, and the device grant for M2 agents. Don't hand-roll the protocol.
- Sign tokens with a KMS asymmetric key (RS256), so the private key never sits in the Lambda. The builder first confirms that `zitadel/oidc` accepts a KMS-backed `crypto.Signer`. If it doesn't, the key is kept in Secrets Manager instead.

## Plan

Four steps, with one cut-over. There is no interim issuer, no dual-write and no staged rollout.

1. **Build** the auth service: a Go Lambda with DynamoDB and KMS, in its own repo.
   - Authorization code with PKCE and refresh tokens, for the SPAs and lets-roll.
   - Client credentials, for service clients.
   - A sign-in page and password reset (email through SES).
   - Impersonation as token exchange, because lets-roll's support impersonation has to keep working after the cut-over.
   - Agent auth for M2 (the device grant) can follow after the cut-over.
2. **Deploy** it alongside the existing apps. Nothing points at it yet.
   - Its issuer URL is fixed from day one (`https://auth.<env domain>`), because every token carries it.
3. **Migrate** every principal: lets-roll user ids (as `sub`), emails and bcrypt hashes. Register every app and service client.
   - The migration is an idempotent upsert, run once to check it and **again immediately before the cut-over**, so sign-ups or password changes in between aren't lost.
   - With prod nearly idle, no dual-write is needed.
4. **Cut over** once: lets-roll, EB, PT and the SPAs all switch their issuer config to the auth service at the same time.
   - lets-roll becomes a relying party: Devise sign-in is replaced by an OIDC client, and `users` gains `principal_id`.
   - EB deletes `internal/lrident`, and kill sync goes live on service tokens.
   - Everyone signs in again once.

**Rollback (a safety net, not a step).** Point the issuer config back at lets-roll.
- Keep lets-roll's Doorkeeper and Devise deployed and untouched until the cut-over has run through a game night. Then remove them.
- After a rollback, tokens from the auth service are rejected, so users sign in again.
- Accounts created in the auth service after the cut-over don't exist in lets-roll, which is acceptable at our scale.

**No shared Postgres.** The auth service never reads lets-roll's database. Reading it would put the Lambda in the VPC, which needs a NAT gateway or interface endpoints, against the "no new NAT unless earned" rule. It would also share a database across services, which the extraction roadmap lists as an anti-pattern.
Principals are exported once and seeded into Dynamo instead.

### DynamoDB notes

- **Unique email.** Use one `TransactWriteItems`: put `PRINCIPAL#<sub>` and put `EMAIL#<normalised email>`, each with `attribute_not_exists`.
  - An email change deletes the old claim item and puts the new one in the same transaction.
  - Normalise by lowercasing and trimming. Don't strip dots or `+` tags.
- **TTL.** Refresh tokens, revocations and invites carry a TTL.
  - TTL deletion is lazy and can take days, so every read compares the expiry itself and never relies on the item being gone.

## Service tokens (`we5k.2.6`, `lets-roll-f6e`)

The auth service is the `we5k.2.6` fix. Every service call uses a client-credentials JWT that it issues.
This replaces both the X-LR-Identity STS proof and 521-architect's earlier IAM suggestion: one mechanism for every service call.

- **Clients and tokens.**
  - Each service has its own client.
  - Tokens are scoped per route (`eb:kill-report`, `eb:pull`, `lr:eb-push`).
  - `exp` is at most 5 minutes. Verifiers allow 30 s of clock skew.
  - The caller caches its token until close to expiry.
- **Service audiences are distinct from SPA client ids**, for example `encounter-builder-<env>-svc` versus `encounter-builder-web`. A service token's `sub` is a client id, not a user, and GM routes treat `sub` as the user. So GM routes must never accept a service audience, and service routes must never accept a user audience.
- **Headers.** Service tokens always travel in `X-Service-Token` and user tokens in `X-Access-Token`. `Authorization` is never used, because CloudFront OAC overwrites it.
- **Secrets.** Each service's client secret lives in Secrets Manager. EB holding one is new.
- **lets-roll → EB (kill reports, the `.2.2` pull).**
  - EB verifies locally with a second verifier, separate from the GM one. It requires `aud` = `encounter-builder-<env>-svc` and the route's scope.
  - The service issuer is configured as `SVC_OIDC_ISSUER`, which defaults to `OIDC_ISSUER`.
- **EB → lets-roll (the push).** The push carries EB's service token in `X-Service-Token` and the GM's bearer in `X-Access-Token`. lets-roll verifies both locally, then checks GM of the game.
- **What this fixes.** A forged token fails a local signature check, with no outbound call, so the amplification is gone.
  - An unknown `kid` is a plain 401, with no refetch. EB's JWKS cache has a 15-minute minimum refresh.
  - Each forgery still costs one Lambda invocation. Reserved concurrency can cap that if needed.
- **What it gives up.** A bearer token is not bound to the method, path or body, as the `#12` binding was.
  - Per-route scope plus the 5-minute `exp` limits a stolen token to one kind of call for a few minutes.
  - DPoP can add that binding later if ever needed.
- **Effect on lets-roll's chain.**
  - `jlm`, `6ru` and `dur` are unchanged.
  - `f6e` becomes a JWT check instead of an STS proof, which is less code.
  - The lets-roll→EB client becomes "fetch a client-credentials token, send it".
- **Timing.** Kill sync stays dark until the cut-over.

## Edge (ruled 2026-11-13)

The auth service's Function URL uses `authorization_type = NONE` with no OAC, and sits behind CloudFront.
This is a deliberate exception to the cluster's OAC-everywhere pattern.

**Why OAC doesn't fit an IdP.** With OAC in front of a Lambda Function URL, every POST and PUT must carry `x-amz-content-sha256` of its body, because CloudFront doesn't compute it.
- Standard OAuth clients POSTing to `/token` don't send that header.
- A browser's HTML sign-in form can't send it at all.
- OAC also overwrites `Authorization`, which breaks `client_secret_basic` and Bearer on userinfo, but the body-hash rule alone rules it out.

**Conditions:**
1. **CloudFront stays in front.** The issuer is the custom domain `https://auth.<env domain>`, fixed from day one.
   - The raw Function URL host is never the issuer, never appears in discovery, and is never given to clients.
2. **Origin verify.** CloudFront sends a secret `X-Origin-Verify` header. The app rejects any request without it, with a constant-time compare, before doing anything else. The value lives in Secrets Manager, and the prod value is Devon's.
3. **Brute-force protection before prod.**
   - Throttle failed sign-ins and failed client auth per account, client and IP, using DynamoDB counters with a TTL.
   - A WAF rate rule on `/token` and the sign-in page is recommended on top. It costs about $6/mo per env, and the cost is Devon's call.
4. **Reserved concurrency** on the Lambda caps what a flood can cost.
5. **Standards as written.** Support `client_secret_basic` and `client_secret_post`, and accept Bearer in `Authorization` on userinfo.
6. **Resource servers keep OAC.** EB, PT and lets-roll are unchanged, so tokens still travel in `X-Service-Token` and `X-Access-Token` there.

## Issuer is not the lets-roll API

EB and PT both use `OIDC_ISSUER` as lets-roll's `/api/v1` base URL today.
- EB uses it in `letsroll.New(issuer)` and in the combat push.
- PT uses it in `service/internal/api/config.go`.

The two SPAs do the same with `VITE_OIDC_AUTHORITY`: `encounter-builder-web` and `party-treasure-web` both call `${authority}/api/v1/…` in `src/api/letsroll.js`.

At the cut-over the issuer stops being lets-roll, so after the cut-over these calls would go to the auth service.
- Each API adds `LETS_ROLL_URL`, defaulting to `OIDC_ISSUER`.
- Each SPA adds `VITE_LETS_ROLL_URL`, defaulting to the authority.
- Each app sets the new value explicitly before the cut-over.
- These changes are backwards compatible, so they can merge ahead of the cut-over date.
- lets-roll's CORS must keep allowing the SPA origins (it allows `*.521studios.com` today).
Also at the cut-over, the GM token on the push moves from `Authorization: Bearer` to `X-Access-Token`, so lets-roll's receiving side must accept that header from then on.

## Availability

- An auth outage blocks sign-in and refresh. It does not block verification of tokens already issued.
- Verifiers cache the JWKS and keep serving the cached copy when a fetch fails.
- User access tokens are 15 minutes, so a session lasts at most 15 minutes past the start of an outage.
  - If that's too short for a game night, raise the access TTL.
  - Don't make verification depend on the auth service.
- Key rotation: issuers publish a new key in the JWKS at least 15 minutes before signing with it, since EB caches the JWKS for at least 15 minutes. Keep the old key published until every token it signed has expired.

## Where it sits

- **The auth service is built now.** It's the `we5k.2.6` fix, and it belongs to `qxlz`. Under the freeze it is the heavy line on ares until the cut-over.
- **M4 (prod cut-over)** includes the auth cut-over if it's ready in time. Otherwise M4 ships on lets-roll's existing provider, and the auth cut-over follows. Both work, because issuers are config.
- **M5, after M4**, is the Rails retirement, in the order below.

## Who builds it

**Recommendation: a new repo, `521-studios/auth-service`, with a new steward session, `auth-service`.**
- It's a long-lived, security-critical service, and it should have its own steward like every other repo in TEAM.md.
- encounter-builder is the alternative. It has the most context on this problem and already runs Go Lambda + DynamoDB + an OIDC verifier. But that would put the issuer and its largest consumer under one session, and pause EB's own line anyway.

**Order under the freeze (one heavy line):**
1. `auth-service` builds and deploys (steps 1–2). encounter-builder and lets-roll take only light work, such as reviewing the client contracts.
2. lets-roll builds its relying-party switch and `f6e` against the deployed service. EB builds its service verifier and drops `lrident`. Both sit behind config, so nothing changes until the cut-over.
3. Migrate (step 3), then cut over (step 4) in one window, with every steward on hand.
4. encounter-builder and lets-roll resume their own lines.

## What Devon must do

- **Approve creating the repo** `521-studios/auth-service`, and the new session.
- **Prod applies**: the auth service's Lambda, DynamoDB, KMS key, and the CloudFront route or domain in infra-frontend. Staging applies are workers'.
- **Prod secrets**: the client secrets for EB, lets-roll and PT, in Secrets Manager.
- **SES**: verify the sending domain and request production access, for password-reset email. Pick the sender address.
- **The prod migration**: exporting lets-roll's users (ids, emails, bcrypt hashes) from the prod database into Dynamo. It handles credentials, so it's Devon's to run or approve.
- **The cut-over window**: pick it, and confirm the game-night rule before Doorkeeper is removed.

## Rails retirement order

Pieces leave in order of how many apps consume them (the extraction rule) and how risky they are.

1. **Auth and principals** (this doc). Impersonation moves with it, as token exchange in the auth service.
2. **Games and membership** (`gonf`). EB and PT both read them (`fcu0.2`), so there are several consumers.
3. **Quantum dice.** It's standalone, and a design exists (`~/521Studios/quantum-dice-fairness-system.md`).
4. **Characters, shortcuts and variables.**
5. **Rolls, initiative, live combat and realtime** (ActionCable, Pixels). These are the core and the largest piece; they need a websocket plan (for example, API Gateway WebSockets). They go last.

When the last piece leaves, lets-roll becomes a static SPA over these services.

## Decisions for Devon

Decided on 2026-11-13: build on `zitadel/oidc`, no shared Postgres, the auth service now, and M5 after M4.

Still open: who builds it. 521-architect recommends a new repo and a new session.
