# Engineering Standards

## Universal Standards

These apply across all 521 Studios repositories regardless of language or framework.

### Every Commit Has Green CI

No exceptions. CI must pass before merge. If CI is broken on main, fixing it is the top priority.

### Rigorous Review

Every change gets reviewed — by a human, an agent reviewer, or both. The goal is correctness first, then consistency, then clarity. Reviews check for:

- Correctness (does it do what it claims?)
- Security (OWASP top 10, no credential leaks, no injection vectors)
- Test coverage for new functionality
- Consistency with repo conventions (see per-repo sections below)

### Test-Driven Development

The direction of travel. Not every repo is there yet, but new work should lead with tests where practical. TDD is especially important for:

- 521-engine (DAG resolution correctness is critical)
- pfsrd2-data-api (search accuracy, template application)
- lets-roll (dice rolling, encounter logic)

### Diffability

A recurring design principle: optimize output formats for human review via `git diff`.

- **pfsrd2-web:** One HTML tag per line (xmllint + newline after `>`)
- **pfsrd2-data:** JSON keys sorted alphabetically
- **Terraform:** `terraform fmt` enforced

If you're generating output that humans need to review, make it diffable.

## Per-Repo Standards

### lets-roll (Ruby/Rails + React)

| Tool | Standard |
|------|----------|
| Ruby | 3.3.10 (rbenv) |
| Rails | 7.1 |
| Node | 24.x (.nvmrc) |
| Linting | Rubocop (pre-commit hook on staged Ruby files) |
| Tests | Minitest + Capybara + Selenium |
| Frontend | React 18, Redux 5, Material-UI 4.x |
| Bundler | Shakapacker 9.x (webpack 5) |
| Issue tracking | beads (`bd` CLI) |

### pfsrd2-data-api (Go)

| Tool | Standard |
|------|----------|
| Go | 1.24 |
| Formatting | `gofmt` (CI enforced) |
| Tests | `go test ./...` |
| Dependencies | chi/v5 (router), modernc.org/sqlite (pure Go, no cgo) |
| Agent reviewers | test-coverage, openapi-spec, error-handling, complexity, sql-injection, clarity |

### pfsrd2-parser (Python)

| Tool | Standard |
|------|----------|
| Python | 3.12 |
| Formatting | black, ruff |
| Dependencies | beautifulsoup4, lxml, jsonschema |
| Error policy | Fail fast — exceptions propagate, zero errors is the only acceptable state |
| Output validation | Every JSON file validated against schema before write |

### pfsrd2-automation (Python)

| Tool | Standard |
|------|----------|
| Python | 3.12+ |
| Package manager | uv |
| Dependencies | boto3, SQLite |

### pfsrd2-display (JavaScript)

| Tool | Standard |
|------|----------|
| Node | 20+ |
| React | 18+ (peer dependency) |
| Build | Rollup 4 (CJS + ESM outputs) |
| Dev server | Vite 6 |
| Tests | Node built-in test runner (no Jest/Vitest) |
| CSS | BEM-style under `.Monster__*` |

### 521-engine (JavaScript)

| Tool | Standard |
|------|----------|
| Node | 22+ |
| Dependencies | Zero (pure JS, stdlib only) |
| Tests | Node built-in test runner |
| Methodology | TDD |

### infra / infra-frontend (Terraform)

| Tool | Standard |
|------|----------|
| Terraform | 1.6+ (infra), 1.9+ (infra-frontend) |
| AWS Provider | ~> 6.0 |
| Formatting | `terraform fmt` (pre-commit hook + CI check) |
| Validation | `terraform validate` (pre-commit, no backend) |
| State | S3 + DynamoDB locking per environment |
| Auth | GitHub OIDC — no stored AWS credentials |

### servitor (Go)

| Tool | Standard |
|------|----------|
| Go | 1.25+ |
| Dependencies | stdlib only — no external deps |
| Formatting | `gofmt` + `go vet` |
| Error handling | Wrap with context, no naked returns |
| Logging | Structured to stderr with timestamps and component tags |
| Locking | Cross-process file locks with stale PID detection |

## Security Standards

- No secrets in code. Use AWS Secrets Manager, environment variables, or SSO.
- GitHub Actions uses OIDC federation — no stored AWS credentials.
- S3 buckets block public access by default.
- Aurora in private subnets, accessible only from ECS security group.
- ECS tasks use scoped IAM roles (least privilege).
- pfsrd2-data-api is read-only, no auth required (public game rules data).
- lets-roll uses Devise for user auth, private tokens for API access.

## Dependency Management

- **Renovate Bot** manages automated dependency updates across Terraform repos.
- lets-roll uses Bundler (Ruby) + Yarn (Node).
- Go repos use stdlib where possible; external deps are minimal and deliberate.
- Python repos use `uv` for package management.
