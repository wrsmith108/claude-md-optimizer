# Monorepo Agent Instructions

## Overview

This is a Python monorepo containing three services: `payments`, `auth`, and `notifications`. Each service is a separate FastAPI application deployable independently. Shared utilities live in `packages/common`.

## Tech Stack
- Language: Python 3.12
- Framework: FastAPI + Pydantic v2
- Database: PostgreSQL (per-service) via SQLAlchemy 2.0 async
- Migrations: Alembic
- Task queue: Celery + Redis
- Testing: pytest + pytest-asyncio + httpx
- CI: GitHub Actions
- Deployment: AWS ECS via Terraform

## Quick Commands

```bash
# Install everything (uses uv workspaces)
uv sync

# Start a specific service in dev mode
uvicorn services/payments/app.main:app --reload --port 8001
uvicorn services/auth/app.main:app --reload --port 8002
uvicorn services/notifications/app.main:app --reload --port 8003

# Run all tests
pytest

# Run tests for one service
pytest services/payments/
pytest services/auth/

# Run with coverage
pytest --cov=services --cov-report=html

# Lint + format
ruff check . && ruff format .

# Type-check
mypy services/ packages/

# Database migrations (run from service directory)
alembic upgrade head           # apply migrations
alembic revision --autogenerate -m "add payment status"  # create migration
alembic downgrade -1           # rollback one migration
```

## Code Conventions
- Type annotations required on all function signatures
- Pydantic v2 models for all request/response schemas — no raw dicts crossing API boundaries
- SQLAlchemy models in `app/models/`, Pydantic schemas in `app/schemas/`
- Async everywhere: use `async def` for route handlers and DB operations
- No `session.execute(text(...))` with user input — use ORM queries
- Service layer in `app/services/` — business logic never in route handlers
- Dependency injection via FastAPI `Depends()` for DB sessions, auth, etc.
- Errors: raise `HTTPException` in route handlers; use custom exception classes in service layer

## PR & Commit Rules
- Conventional Commits: `feat(payments): add refund endpoint`, `fix(auth): handle expired tokens`
- PRs must include tests for new endpoints
- Use draft PRs for work in progress
- All PRs require 1 review + passing CI
- Squash-merge only

## Payments Service (services/payments/)

The payments service handles charge creation, refunds, and webhook processing from Stripe.

### Key Concepts
- A `Payment` record is created when a checkout session begins; status transitions: `pending → succeeded | failed | refunded`
- Webhook events from Stripe update Payment status atomically — idempotency key is the Stripe event ID
- Refunds must be initiated server-side via Stripe API — never trust client-supplied refund amounts
- PCI compliance: never log card details; do not store full card numbers anywhere

### Payments-Specific Commands
```bash
# Run Stripe webhook listener locally (requires Stripe CLI)
stripe listen --forward-to localhost:8001/webhooks/stripe

# Test a specific webhook event
stripe trigger payment_intent.succeeded
```

### Payments Database Schema
- `payments` — `id`, `stripe_payment_intent_id`, `user_id`, `amount_cents`, `currency`, `status`, `metadata`, `created_at`, `updated_at`
- `refunds` — `id`, `payment_id`, `stripe_refund_id`, `amount_cents`, `reason`, `status`, `created_at`
- `webhook_events` — `id`, `stripe_event_id`, `type`, `processed_at`, `raw_payload` (for idempotency)

### Payments Testing Notes
- Use `pytest-recording` for Stripe API calls — cassettes in `tests/cassettes/`
- Always use the `stripe-signature` fixture to simulate valid webhook signatures in tests
- Payment status state machine is tested in `tests/unit/test_payment_state.py`

## Auth Service (services/auth/)

The auth service issues JWT access tokens and manages user identity. All other services validate tokens against this service's JWKS endpoint.

### Key Concepts
- Access tokens: 15 min TTL, signed RS256
- Refresh tokens: 30 day TTL, stored in `refresh_tokens` table, single-use (rotate on use)
- JWKS endpoint: `GET /auth/.well-known/jwks.json` — cache for 5 min in other services
- Password resets: time-limited signed URLs (1 hour), consumed on first use
- Rate limits: 10 login attempts per email per 15 min (Redis sliding window)

### Auth Database Schema
- `users` — `id`, `email`, `hashed_password`, `name`, `is_active`, `created_at`, `updated_at`
- `refresh_tokens` — `id`, `user_id`, `token_hash`, `expires_at`, `used_at`, `revoked_at`
- `password_resets` — `id`, `user_id`, `token_hash`, `expires_at`, `used_at`

### Auth Testing Notes
- Use the `auth_client` fixture from `packages/common/testing/fixtures.py` — it handles token issuance
- Test token expiry with `freezegun` — don't sleep in tests
- Key pair for tests is in `tests/fixtures/test_keys/` — never use production keys in tests

## Notifications Service (services/notifications/)

Sends emails and in-app notifications. Consumes events from a Celery queue; never called directly by other services.

### Key Concepts
- Email via SendGrid; in-app notifications stored in `notifications` table
- Events published to queue by other services using `packages/common/events.py`
- Notification templates in `services/notifications/templates/` (Jinja2)
- Unsubscribe tokens are signed with HMAC-SHA256 — validate before processing

### CI Pipeline Details

**ci.yml** — runs on every PR:
1. `uv sync`
2. `ruff check . && ruff format --check .`
3. `mypy services/ packages/`
4. `pytest --cov=services --cov-report=xml`
5. Upload coverage to Codecov
6. Coverage < 85% fails pipeline

**integration.yml** — runs on push to `main`:
1. Spin up `docker compose -f docker-compose.ci.yml up -d` (postgres, redis)
2. `alembic upgrade head` per service
3. `pytest tests/integration/ -v`
4. Tear down docker services

**deploy.yml** — runs on tag `v*`:
1. Build Docker images per service
2. Push to ECR: `aws ecr get-login-password | docker login ...`
3. Update ECS task definitions via Terraform
4. ECS rolling update; health check must pass within 5 min

## Troubleshooting

### Alembic "Target database is not up to date"
Another migration was applied elsewhere. Run `alembic history` to see the full chain, then `alembic upgrade head`.

### "Connection pool exhausted" under load
Check `SQLALCHEMY_POOL_SIZE` (default 5) and `SQLALCHEMY_MAX_OVERFLOW` (default 10). For tests, use `NullPool` to avoid connection leaks: set `TESTING=true` in `.env.test`.

### Celery tasks not processing
1. Check Redis is running: `redis-cli ping`
2. Check worker is up: `celery -A packages.common.celery_app inspect active`
3. Check queue: `celery -A packages.common.celery_app inspect reserved`
4. Retry failed tasks: `celery -A packages.common.celery_app call <task_name> --args='[]'`

### JWT validation errors in downstream services
1. Check JWKS cache — it may be stale. Force refresh by restarting the consuming service or clearing its cache.
2. Verify clock skew < 30 s between services (NTP sync issue)
3. Check `AUTH_SERVICE_URL` env var points to correct auth service instance

### Docker image build failures
- Multi-stage build; `uv export --no-dev` generates requirements. If this fails, check `pyproject.toml` for dependency conflicts.
- Layer cache: `docker build --no-cache` if you suspect stale cache

## Infrastructure & Deployment

### Local Environment
Docker Compose for local dev:
```yaml
services:
  postgres-payments:
    image: postgres:16
    environment: { POSTGRES_DB: payments_dev, POSTGRES_USER: dev, POSTGRES_PASSWORD: devpass }
    ports: ["5432:5432"]
  postgres-auth:
    image: postgres:16
    environment: { POSTGRES_DB: auth_dev, POSTGRES_USER: dev, POSTGRES_PASSWORD: devpass }
    ports: ["5433:5432"]
  redis:
    image: redis:7
    ports: ["6379:6379"]
```

Start all services: `docker compose up -d`
Stop: `docker compose down`

### Environment Variables

Each service reads its config from environment variables via `packages/common/config.py` (pydantic-settings). Copy `.env.example` to `.env` in each service directory:

```
# services/payments/.env.example
DATABASE_URL=postgresql+asyncpg://dev:devpass@localhost:5432/payments_dev
REDIS_URL=redis://localhost:6379/0
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
AUTH_SERVICE_URL=http://localhost:8002
SENTRY_DSN=https://xxx@sentry.io/xxx

# services/auth/.env.example
DATABASE_URL=postgresql+asyncpg://dev:devpass@localhost:5433/auth_dev
REDIS_URL=redis://localhost:6379/1
JWT_PRIVATE_KEY_PATH=./keys/private.pem
JWT_PUBLIC_KEY_PATH=./keys/public.pem
SENDGRID_API_KEY=SG.xxx
FRONTEND_URL=http://localhost:3000

# services/notifications/.env.example
DATABASE_URL=postgresql+asyncpg://dev:devpass@localhost:5433/notifications_dev
REDIS_URL=redis://localhost:6379/2
SENDGRID_API_KEY=SG.xxx
```

### AWS / ECS Deployment

Terraform configuration lives in `infra/`. Each service has its own ECS service and task definition.

```bash
# Deploy a single service (from infra/)
terraform workspace select staging
terraform apply -target=module.payments_service

# View running tasks
aws ecs list-tasks --cluster prod-cluster --service-name payments
aws ecs describe-tasks --cluster prod-cluster --tasks <task-arn>

# Force new deployment (rolling update)
aws ecs update-service --cluster prod-cluster --service payments --force-new-deployment
```

Health check endpoint for all services: `GET /health` returns `{"status": "ok", "version": "<git-sha>"}`.

Log groups in CloudWatch: `/ecs/payments`, `/ecs/auth`, `/ecs/notifications`. Tail logs:
```bash
aws logs tail /ecs/payments --follow
```

### Rollback Procedure
1. Identify last good image tag from ECR (`aws ecr describe-images --repository-name payments`)
2. Update task definition to use that image tag: `terraform apply -var="payments_image_tag=<tag>"`
3. Force redeployment: `aws ecs update-service --force-new-deployment ...`
4. Verify health check passes within 5 min

## Onboarding

New developer setup:
1. Clone repo
2. Install `uv`: `curl -LsSf https://astral.sh/uv/install.sh | sh`
3. `uv sync` (installs all workspace dependencies)
4. Copy `.env.example` to `.env` in each service directory
5. `docker compose up -d`
6. Run migrations per service: `cd services/payments && alembic upgrade head`
7. Seed test data: `uv run python scripts/seed_dev_data.py`
8. Start services: open 3 terminals, one per service
9. Verify: `curl http://localhost:8001/health`, `curl http://localhost:8002/health`

For AWS access requests, post in `#dev-access` Slack channel with your IAM username.
