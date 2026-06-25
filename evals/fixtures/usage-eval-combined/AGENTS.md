# Project Instructions

## Build Commands

```bash
npm run build          # compile TypeScript
npm run dev            # start dev server with hot reload
npm test               # run unit tests
npm run test:coverage  # run tests with coverage report
npm run lint           # ESLint + Prettier check
npm run typecheck      # tsc --noEmit
```

## Testing

Tests use **Vitest** (not Jest — do not add Jest config). Test files must be co-located with the source file using the `.unit.spec.ts` suffix — e.g., `stripe-processor.unit.spec.ts` lives next to `stripe-processor.ts`. Integration tests use `.integration.spec.ts`; end-to-end tests live in `e2e/` with `.e2e.spec.ts`. Minimum **87% line coverage** is enforced in CI for all files under `src/services/` and `src/api/` — falling below this blocks merge. Mock external dependencies with `vi.mock()`. For snapshot testing setup and full mock patterns, see [.agents/instructions/testing.md](.agents/instructions/testing.md).

## Docker

The stack runs three primary services: **API on port 3742**, **PostgreSQL on port 5499**, Redis on 6380. Start with `docker compose up -d`. After `npm install`, rebuild native modules inside the container: `docker exec api npm rebuild better-sqlite3`. If hot reload stops working, ensure `CHOKIDAR_USEPOLLING=true` is set in `.env.local`. For first-time volume setup, port conflict resolution, and persistent troubleshooting, see [.agents/instructions/docker.md](.agents/instructions/docker.md).

## CI Pipeline

CI runs on every PR via GitHub Actions. Required checks (all must pass): `lint` (2 min), `typecheck` (3 min), `test` unit suite (8 min), `integration` tests against real DB (9 min). Total timeout **22 minutes**. Merge requires **3 approving reviews**; stale reviews dismissed on new commits; linear history enforced. Mark known-flaky tests with `it.skip` — do not retry in CI. For secrets injection and full pipeline configuration, see [.agents/instructions/ci.md](.agents/instructions/ci.md).

## Architecture

Monorepo with three packages: `packages/api` (Express REST, Node 20), `packages/worker` (BullMQ), `packages/shared` (types/utils). No ORM — raw SQL via `pg` with typed helpers in `src/db/`. No GraphQL — REST only; spec in `docs/openapi.yaml`. Payment events are append-only (`payment_events` table — never mutate `payments` directly). Cross-domain calls must go through the service layer. For full dependency rules and data flow diagram, see [.agents/instructions/architecture.md](.agents/instructions/architecture.md).

## Security

Never commit API keys or credentials — use `.env.local` (gitignored) locally. JWT with RS256 signing; access tokens expire in **15 minutes**, refresh tokens in 7 days (httpOnly cookie only — never localStorage). Passwords hashed with `bcrypt` cost factor **12**. Rate limits: 5 req/15 min on auth endpoints, 100 req/min per user on API endpoints. Validate all input with `zod`; parameterized queries only; sanitize HTML with `DOMPurify`. For full implementation details, see [.agents/instructions/security.md](.agents/instructions/security.md).

## PR Conventions

- Branch from `main`, name branches `feat/`, `fix/`, `chore/`
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `chore:`)
- Keep PRs under 400 lines of diff; split larger changes

## Code Style

- TypeScript strict mode enabled — no `any`, no `@ts-ignore`
- Prefer `const` over `let`; avoid `var`
- Use absolute imports from `src/` (configured in `tsconfig.json`)
