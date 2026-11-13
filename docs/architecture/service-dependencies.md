# Service Dependency Map

## Runtime Dependencies

These are live connections between services at runtime.

```
                    ┌─────────────┐
                    │  CloudFront │
                    │ (single     │
                    │  distro)    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
         ┌─────────┐ ┌─────────┐ ┌──────────┐
         │lets-roll │ │pfsrd2   │ │pfsrd2    │
         │(ECS/ALB)│ │data-api │ │images    │
         │         │ │(Lambda) │ │(S3)      │
         └────┬────┘ └────┬────┘ └──────────┘
              │            │
         ┌────┼────┐       │
         │    │    │       │
         ▼    ▼    ▼       ▼
       Aurora Valkey    S3 (pfsrd2
       (RDS) (cache)    SQLite DB
                        + JSON)
```

### lets-roll → pfsrd2-data-api

- **Type:** HTTP API calls from browser (client-side)
- **Path:** CloudFront `/api/pfsrd2/*` → Lambda Function URL
- **Data flow:** lets-roll's React frontend fetches monster data on demand. No server-side calls. Nothing stored in lets-roll's DB — monsters exist only in client memory.
- **Future:** Encounters will store reference IDs (game_id) but not full stat blocks.

### lets-roll → Aurora PostgreSQL

- **Type:** Direct TCP (port 5432) via private subnet
- **Data:** Users, games, characters, rolls, shortcuts, variables, encounters, initiative queues
- **Credentials:** Secrets Manager, injected via ECS task definition

### lets-roll → Valkey (ElastiCache)

- **Type:** Redis protocol (port 6379) via private subnet
- **Purpose:** Action Cable (WebSocket pub/sub) + background jobs
- **URL:** Injected as `REDIS_URL` env var

### pfsrd2-data-api → S3

- **Type:** AWS SDK (GetObject, HeadObject)
- **Data:** SQLite index DB (`db/pfsrd2.db`), JSON entries, images
- **Caching:** ETag-based. Background watcher checks hourly. Atomic DB swap on change.

### CloudFront → Origins

| Path | Origin | Protocol |
|------|--------|----------|
| `/cable` | ALB | HTTP (WebSocket upgrade) |
| `/assets/*` | ALB | HTTP |
| `/api/pfsrd2/*` | Lambda Function URL | HTTPS (OAC + SigV4) |
| `/pfsrd2/images/*` | S3 | HTTPS (OAC) |
| `/*` default | ALB | HTTP |

CloudFront terminates TLS. ALB receives HTTP only.

## Encounter / Treasure Cluster

Each SPA has its own CloudFront edge (infra-frontend `encounter-builder-cf`,
`party-treasure-cf`). Its own API sits behind it, along with pfsrd2-data-api,
both through OAC.

| From | To | How |
|---|---|---|
| SPAs, sibling APIs | lets-roll OIDC | Bearer JWT (RS256). APIs verify offline after fetching JWKS. Discovery is lazy, so a lets-roll outage gives 401s, not a crash. Behind OAC the bearer travels in `X-Access-Token`, because `Authorization` carries SigV4. |
| encounter-builder-api, party-treasure-api | lets-roll `GET /api/v1/games/:id` | Caller's bearer forwarded, to decide GM vs member. |
| encounter-builder-api | party-treasure-api `POST /parties/{id}/releases` | **Server-to-server**, SigV4. Identity is matched exactly against `SERVICE_RELEASE_ROLE_ARN`. Idempotent on the client ULID. |
| SPAs | pfsrd2-data-api | Same-origin path on each edge. Clients resolve ContentRefs; the APIs never do. |
| encounter-builder-web | encounter-builder S3 maps bucket | Presigned POST (≤15 MB, png/jpeg/webp, tagged `state=pending` until saved; untagged-on-save, pending expire after 1 day). Served through the encounters edge with uuid keys. Bucket in encounter-builder-api terraform; origin, OAC and bucket policy in infra-frontend. |

ContentRef, the shared shape for "a monster" or "an item", is a
cross-service contract: [content-ref.md](content-ref.md).

**Divergence from plan.** [encounter-treasure-cluster.md](encounter-treasure-cluster.md)
§5 planned the release as a browser-driven call, keeping the "browser is the
integration point" rule. What shipped is a SigV4 service push from
encounter-builder-api. That is the cluster's one server-to-server edge, and a
deliberate exception to "Shared Nothing" below. Devon's reason (2026-11-10):
moving treasure into a party is "easier to be secure if it's a tested and
validated backend operation" than a browser handoff.

**Planned:** two more server-to-server edges between encounter-builder-api and
lets-roll. They carry the encounter pull and push, and the kill sync. The push
is `POST /api/v1/games/:id/eb_encounters`, authenticated with a presigned STS
`GetCallerIdentity` request, because lets-roll's ALB has no IAM authorizer. See
[encounter-lifecycle-bridge.md](encounter-lifecycle-bridge.md) (`we5k.2`).

**Membership's future home.** Games, members and characters live in lets-roll
today. party-treasure-api reads them from lets-roll `/games/:id/members`
(`fcu0.2`). They are to move out of lets-roll, behind either a shared service
any app can use, or DynamoDB (Devon, 2026-11-11; `gonf`, Later).

## Build-Time Dependencies

These are consumed during CI/CD or development, not at runtime.

### Terraform Remote State Chain

```
infra (baseline)
  ↓ provides: vpc_id, subnet_ids, ecs_cluster, alb_security_group_id, route53_zone_id, etc.
lets-roll terraform
  ↓ provides: ALB (owned here since infra-1.2); its DNS is consumed by infra-frontend
pfsrd2-data-api terraform
  ↓ provides: lambda_function_url, s3_bucket_name, s3_bucket_regional_domain
infra-frontend
  reads from: infra (zone_id) + pfsrd2-data-api (lambda URL, S3 bucket)
```

### Data Pipeline Dependencies

```
pfsrd2-web (HTML source)
  ↓ read by
pfsrd2-parser (Python)
  ↓ writes to
pfsrd2-data (JSON files)
  ↓ read by
pfsrd2-automation (indexer)
  ↓ uploads to
S3 bucket (owned by pfsrd2-data-api terraform)
  ↓ downloaded by
pfsrd2-data-api (at cold start + hourly)
```

### pfsrd2-automation → pfsrd2-data-api

- pfsrd2-data-api's Terraform exports an `indexer_iam_policy_arn`
- pfsrd2-automation's CI assumes a role with this policy to write to S3
- The S3 bucket is owned by pfsrd2-data-api's Terraform

### lets-roll → pfsrd2-data-api (development)

- `docker-compose.yml` in lets-roll starts pfsrd2-data-api as a sidecar service
- Local development uses the same API as production, just pointed at staging S3

## Component Library Dependencies

### pfsrd2-display

- **Peer dependency:** React 18+
- **Runtime dependency:** markdown-it (Markdown + PF2e action icon rendering)
- **API dependency:** Consumes JSON from pfsrd2-data-api (search, entries, templates)
- **Consumers:** None yet (extracted from lets-roll, not re-integrated)
- **Published as:** `@521studios/pfsrd2-display` (private npm package)

### 521-engine

- **Dependencies:** None (pure JS, zero deps)
- **Planned consumers:** lets-roll (encounters), Monster Lab, pfsrd2-data-api (embedded in Go)
- **Data source:** Will consume pfsrd2-data JSON, wrapping static content with resolution logic

## Cross-Cutting Concerns

### Authentication

- **lets-roll users:** Devise (email/password)
- **API access:** Private tokens (per-user, per-game, per-character)
- **AWS CI/CD:** GitHub OIDC federation
- **AWS developer access:** AWS SSO profiles
- **pfsrd2-data-api:** No auth (public read-only API)

### Observability

- **lets-roll:** CloudWatch logs (ECS), Mixpanel (user analytics)
- **pfsrd2-data-api:** CloudWatch logs (Lambda, 14-day retention), structured slog
- **Infrastructure:** CloudWatch Container Insights, ALB health checks
- **Smoke tests:** Playwright E2E in CI, screenshots saved as artifacts

### Shared Nothing

Services share infrastructure (VPC, ECS cluster) but not application state. lets-roll and pfsrd2-data-api have no direct server-to-server communication — the browser is the integration point. This is intentional: services can be developed, deployed, and scaled independently.
