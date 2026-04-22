---
description: "Monorepo layout (packages/api, packages/worker, packages/shared), no ORM, no GraphQL, append-only payment events, cross-domain call rules"
loading_strategy: "lazy"
---

# Architecture

## Service Layout

Monorepo with three main packages:

- `packages/api` — Express REST API (Node.js 20)
- `packages/worker` — BullMQ job processor
- `packages/shared` — Shared types and utilities

## Data Flow

```
Client → API (packages/api) → PostgreSQL (primary)
                            → Redis (cache + queues)
                            → Worker (async jobs via BullMQ)
```

## Domain Boundaries

Payments, auth, and notifications are separate domain modules under `src/services/`. Cross-domain calls must go through the service layer, never directly between repositories.

## Key Design Decisions

- **No ORM**: Raw SQL via `pg` library with typed query helpers in `src/db/`
- **No GraphQL**: REST only; OpenAPI spec lives in `docs/openapi.yaml`
- **Event sourcing**: Payment events are append-only; use `payment_events` table, never mutate `payments` directly

## Dependency Rules

`shared` → no dependencies on `api` or `worker`
`worker` → may import from `shared` only
`api` → may import from `shared` only
