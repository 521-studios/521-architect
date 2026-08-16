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

## Build-Time Dependencies

These are consumed during CI/CD or development, not at runtime.

### Terraform Remote State Chain

```
infra (baseline)
  ↓ provides: vpc_id, subnet_ids, ecs_cluster, alb, route53_zone_id, etc.
lets-roll terraform
  ↓ provides: (self-contained, but ALB DNS consumed by infra-frontend)
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
