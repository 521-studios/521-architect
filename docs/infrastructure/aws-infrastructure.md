# AWS Infrastructure Architecture

## Overview

521 Studios runs on AWS with a three-layer Terraform architecture. Each layer has its own state and deploys independently, connected via `terraform_remote_state` data sources.

```
Layer 1: infra (baseline)
  VPC, Aurora, ECS cluster, S3, Valkey, Route53 zone
       ↓ outputs consumed via remote state
Layer 2: app terraform (per-service)
  lets-roll: ALB, ECS service, ECR, ElastiCache
  pfsrd2-data-api: Lambda, S3 bucket, IAM
       ↓ outputs consumed via remote state
Layer 3: infra-frontend
  CloudFront distribution, ACM certs, DNS records, path-based routing
```

**Deploy order matters.** Each layer reads outputs from the layer below. Deploying out of order causes `terraform plan` failures.

## AWS Accounts

| Account | ID | Purpose |
|---------|-----|---------|
| Production | 242258465702 | Live traffic |
| Staging | 242288027927 | Pre-production |
| Management | 765836143833 | AWS Organizations root |

Authentication: AWS SSO via `521-Staging-Admin` and `521-Production-Admin` profiles. GitHub Actions uses OIDC federation — no stored credentials.

## Layer 1: infra (baseline)

**Repo:** `521-studios/infra`

Shared resources consumed by all services:

| Resource | Details |
|----------|---------|
| VPC | Public subnets (ECS/ALB) + private subnets (RDS/Valkey), 3 AZs in us-east-2 |
| Aurora PostgreSQL 16.8 | Serverless v2, auto-pause to 0 ACU, credentials in Secrets Manager |
| ECS Fargate cluster | FARGATE + FARGATE_SPOT capacity providers, Container Insights |
| ElastiCache Valkey 7.2 | Redis-compatible, for Action Cable + background jobs |
| S3 | Rails ActiveStorage bucket (versioned, private, SSE-S3) |
| Route53 | 521studios.com zone (prod), staging.521studios.com (staging) |
| ECS Task Cleanup Lambda | Kills orphaned non-service tasks >30min old, runs every 15min |

### Network Layout

- **Production CIDR:** 10.0.0.0/16
- **Staging CIDR:** 10.1.0.0/16
- **Public subnets:** ALB, ECS Fargate tasks (publicly routable — no NAT gateway)
- **Private subnets:** Aurora RDS, ElastiCache Valkey
- **Security groups:** ALB (HTTP/HTTPS from internet), ECS (from ALB only), RDS (from ECS on 5432), Valkey (from ECS on 6379)

### Key Outputs

All downstream services consume these via `terraform_remote_state`:
- `vpc_id`, `public_subnet_ids`, `private_subnet_ids`
- `ecs_cluster_arn`, `ecs_cluster_name`
- `alb_arn`, `alb_listener_https_arn`, `alb_security_group_id`
- `task_execution_role_arn`, `task_role_arn`
- `db_cluster_endpoint`, `db_secret_arn`
- `activestorage_bucket_name`
- `valkey_primary_endpoint`, `valkey_port`
- `route53_zone_id`

## Layer 2: App Terraform

Each service manages its own infrastructure in its own repo.

### lets-roll

- ECR repository (scan-on-push, keep last 20 images)
- ALB target group (health check every 30s, 30s drain for WebSockets)
- ECS task definition (512 CPU, 1024 MB RAM)
- ECS service (Fargate, public IPs — no NAT)
- ElastiCache Valkey cluster (Action Cable + jobs)
- Route53 CNAME: `lets-roll-origin.{domain}` → ALB DNS
- Production guard: prevents accidental destruction

### pfsrd2-data-api

- S3 bucket (versioning, AES256, public access blocked)
- Lambda function (ARM64, 512MB, 30s timeout, custom Go runtime)
- IAM roles/policies (Lambda execution, S3 access, indexer write policy)
- CloudWatch log group (14-day retention)
- Does NOT consume infra remote state — uses AWS data sources to look up shared resources by name

## Layer 3: infra-frontend

**Repo:** `521-studios/infra-frontend`

Single CloudFront distribution with path-based routing — one entry point for all public traffic:

| Path | Origin | Caching |
|------|--------|---------|
| `/cable` | ALB | Disabled (WebSocket) |
| `/assets/*` | ALB | Aggressive |
| `/pfsrd2/images/*` | pfsrd2 S3 bucket (OAC) | Aggressive |
| `/api/pfsrd2/*` | Lambda Function URL (OAC + SigV4) | Disabled |
| `/*` (default) | ALB | Disabled |
| 502/503/504 | S3 maintenance page | 60s TTL |

Also owns:
- ACM certificates (us-east-1, required by CloudFront)
- Route53 DNS: lets-roll.org, 521studios.com, www redirect
- SendGrid email authentication records (DKIM, DMARC)
- CloudFront Function for www→apex redirect

## CI/CD Pattern

All three layers follow the same pattern:

1. **PR:** `terraform fmt -check` + parallel plan for staging and production, posted as PR comment
2. **Merge to main:** Auto-apply staging
3. **Production:** Manual `workflow_dispatch` with approval gate
4. **Auth:** GitHub OIDC → assume IAM role, no stored credentials

### lets-roll Deploy Pipeline

1. Terraform apply (ECS task def, ALB, etc.)
2. Docker multi-stage build (Node assets → Ruby gems → runtime image)
3. Push to ECR
4. Run DB migrations in one-shot Fargate task
5. Update ECS service with new task definition
6. Wait for service to stabilize (health checks at `/up`)

## State Management

- S3 backend per environment: `521studios-infra-tfstate-<account-id>`
- DynamoDB locking: `521studios-infra-tfstate-lock`
- State encryption: AES256 (SSE-S3)
- Bootstrap script (`infra/scripts/bootstrap.sh`) handles initial S3 bucket + DynamoDB table + OIDC provider setup

## Cost Discipline

Documented principles in infra-frontend:
- Reuse existing resources (path behaviors vs. new distributions)
- New CloudFront distribution + ACM cert = ~$0.60/month minimum
- Add path behaviors to existing distribution instead of creating new ones
- Aurora Serverless v2 auto-pauses to zero in staging
- FARGATE_SPOT for staging (70% cheaper)

## Developer Access

**Repo:** `521-studios/tools`

Heroku-like CLI utilities for AWS:
- `./db` — Aurora PostgreSQL via SSM port-forward (temporary Fargate tunnel proxy)
- `./console` — Rails console via ECS Exec
- `./logs` — CloudWatch log tailing

All require AWS SSO session. Setup documented in `521-studios/documentation`.
