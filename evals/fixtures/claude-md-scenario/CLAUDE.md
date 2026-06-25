# Project Instructions for Claude

## Tech Stack
- Backend: Node.js 20 + TypeScript 5 + Express 4
- Database: PostgreSQL 16 with Prisma ORM
- Frontend: React 18 + TypeScript + Vite
- Testing: Vitest + Playwright
- CI: GitHub Actions
- Deployment: Render.com

## Quick Commands

```bash
npm install           # install dependencies
npm run dev           # start backend + frontend (hot reload)
npm run build         # production build
npm test              # run all tests
npm run test:unit     # unit tests only (Vitest)
npm run test:e2e      # e2e tests only (Playwright — needs running server)
npm run test:unit -- --coverage   # unit tests with coverage report
npm run typecheck     # run tsc --noEmit
npm run lint          # run ESLint
npm run format        # run Prettier
```

## Database Commands

```bash
npx prisma migrate dev             # apply pending migrations in dev
npx prisma migrate deploy          # apply pending migrations in prod (CI/deploy)
npx prisma db push                 # push schema without migration (dev only, resets data)
npx prisma generate                # regenerate Prisma client after schema change
npx prisma db seed                 # seed development data
npx prisma studio                  # open Prisma Studio GUI
```

## Security Rules
- NEVER commit API keys, secrets, or credentials to the repository
- NEVER log sensitive user data (passwords, tokens, PII)
- Always validate user input server-side — never trust client-only validation
- Use Prisma parameterized queries — never construct raw SQL from user input
- Passwords are hashed with bcrypt cost factor 12 — never store plaintext
- JWT access tokens expire in 1 hour; refresh tokens expire in 30 days
- Rate-limit all auth endpoints: 100 req/15 min per IP (already in middleware)

## Test File Patterns
- Unit test files: `*.test.ts` / `*.test.tsx` — co-located with source
- E2E test files: `tests/e2e/**/*.spec.ts`
- Minimum 80% line coverage required for `src/api/` and `src/services/`
- Mock external HTTP calls with `vi.mock` — no real network in unit tests
- Snapshot tests are discouraged — prefer explicit assertions
- Keep test files under 100 lines; split if needed

## Git Workflow
- Branch from `main`; naming: `feature/JIRA-123-slug`, `fix/JIRA-456-slug`, `chore/slug`
- Conventional Commits: `feat: add avatar upload`, `fix: handle null user`
- Always squash-merge PRs
- PRs require 1 reviewer + passing CI

## Code Style

TypeScript:
- `"strict": true` in tsconfig.json — no exceptions
- Prefer `type` over `interface` for object shapes
- No `any` — use `unknown` or proper generics
- No `!` non-null assertions — handle nullability explicitly
- Use `satisfies` over `as` for type assertions where possible
- Import alias: `@/` maps to `src/`

React:
- Function components only — no class components
- Custom hooks for shared stateful logic
- Co-locate test files: `MyComponent.tsx` + `MyComponent.test.tsx`
- CSS Modules for component styles: `MyComponent.module.css`
- Named exports only — no default exports for components

## Architecture

Layered monolith with clear module boundaries:

```
src/
  api/           # Express route handlers (HTTP layer only)
  services/      # Business logic (no HTTP concerns)
  repositories/  # DB access via Prisma (no business logic)
  middleware/     # Express middleware
  workers/        # Background jobs (BullMQ)
  types/          # Shared TypeScript types
  utils/          # Pure utility functions
  config/         # App config loaded from env vars
```

Dependency rules:
- Route handlers → Services only (never repositories directly)
- Services → Repositories for data access
- Repositories → Prisma only (thin wrappers, no business logic)
- No circular dependencies (enforced by `eslint-plugin-import`)
- All config values come from `src/config/` — no hardcoded values elsewhere

## Database Schema

Prisma models (abbreviated):
- `User` — `id`, `email`, `hashedPassword`, `name`, `avatarUrl`, `createdAt`, `updatedAt`
- `Session` — `id`, `userId`, `token`, `expiresAt`
- `Post` — `id`, `authorId`, `title`, `slug`, `content`, `publishedAt`, `tags`
- `Comment` — `id`, `postId`, `authorId`, `content`, `createdAt`
- `Reaction` — `id`, `postId`, `userId`, `type`

Migration rules:
- Always `npx prisma migrate dev` when schema changes
- NEVER edit committed migration files
- Name migrations descriptively: `20240115_add_user_avatar`
- Seed data in `prisma/seed.ts` — run with `npx prisma db seed`

## Docker / Local Dev

Local development uses Docker Compose (already in repo):

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: devpassword
      POSTGRES_DB: appdb
    ports: ["5432:5432"]
    volumes: [pgdata:/var/lib/postgresql/data]
  redis:
    image: redis:7
    ports: ["6379:6379"]
```

Commands:
```bash
docker compose up -d       # start local services
docker compose down        # stop local services
docker compose logs -f     # tail logs
```

Connecting to dev DB:
- Connection string: `postgresql://dev:devpassword@localhost:5432/appdb`
- GUI access: `npx prisma studio`

## CI/CD Pipeline

**ci.yml** — runs on every PR:
1. `npm ci`
2. `npm run typecheck`
3. `npm run lint`
4. `npm run test:unit -- --coverage`
5. Codecov uploads coverage report
6. Coverage < 80% fails the pipeline

**e2e.yml** — runs on push to `main`:
1. Spin up `docker compose up -d`
2. `npm ci && npm run build`
3. Start server: `node dist/server.js &`
4. Wait for server: `wait-on http://localhost:3000/health`
5. `npm run test:e2e`
6. Upload Playwright HTML report as artifact (always, even on failure)

**deploy.yml** — runs on successful e2e against `main`:
1. Authenticate with Render service account
2. Trigger deploy hook: `curl -X POST $RENDER_DEPLOY_HOOK`
3. Poll deploy status every 30 s until complete or timeout (10 min)
4. Run smoke test: `curl -f https://staging.yourapp.com/health`

**Branch protection rules:**
- `main` is protected; direct pushes blocked
- Required status checks: `ci / typecheck`, `ci / lint`, `ci / test`
- 1 approving review required

## Troubleshooting

### "Module not found" in TypeScript
Usually a path alias issue. Verify both:
- `tsconfig.json`: `"paths": { "@/*": ["./src/*"] }`
- `vite.config.ts`: `resolve: { alias: { '@': path.resolve(__dirname, 'src') } }`
- Vitest `moduleNameMapper` matches tsconfig paths

### Prisma client out of date
Run `npx prisma generate`. Happens after pulling schema changes or updating Prisma version.

### Port already in use
```bash
lsof -i :3000 | grep LISTEN   # find PID
kill -9 <PID>
```
Or use a different port: `PORT=3001 npm run dev`

### Database connection refused
1. Check containers: `docker ps`
2. Start if needed: `docker compose up -d`
3. Verify `DATABASE_URL` in `.env` matches Docker Compose settings
4. Test connection: `psql postgresql://dev:devpassword@localhost:5432/appdb`

### Tests failing in CI but passing locally
1. Check `.env.test` — CI only loads this file
2. Flaky tests: rerun with `--reporter=verbose` to isolate
3. Coverage failures: `npm run test:unit -- --coverage --reporter=verbose`

### Redis / BullMQ errors
Default Redis URL: `redis://localhost:6379` — set `REDIS_URL` in `.env` if different.
Queue names live in `src/workers/queues.ts`. Debug with `LOG_LEVEL=debug npm run dev`.

### E2E timeouts in CI
- Playwright timeout is 30 s by default; increase with `PLAYWRIGHT_TIMEOUT=60000`
- Check screenshots in the uploaded artifact — Playwright captures on failure

## Environment Variables

Copy `.env.example` to `.env` for local dev:

```
DATABASE_URL="postgresql://dev:devpassword@localhost:5432/appdb"
REDIS_URL="redis://localhost:6379"
JWT_SECRET="dev-secret-change-in-production"
JWT_REFRESH_SECRET="dev-refresh-secret-change-in-production"
NODE_ENV="development"
PORT=3000
FRONTEND_URL="http://localhost:5173"
RENDER_DEPLOY_HOOK="https://api.render.com/deploy/srv-xxxx?key=yyyy"
SENDGRID_API_KEY="SG.xxxxx"
AWS_ACCESS_KEY_ID="AKIAXXXXXX"
AWS_SECRET_ACCESS_KEY="xxxxxxxx"
AWS_S3_BUCKET="dev-uploads"
AWS_REGION="us-east-1"
SENTRY_DSN="https://xxx@sentry.io/xxx"
```

Production env vars are managed in Render dashboard — never in code.

## Deployment

### Staging (Render.com)
- Auto-deploys from `main` on successful CI
- URL: `https://staging.yourapp.com`
- `NODE_ENV=staging`

### Production (Render.com)
- Manual deploy required — no auto-deploy from prod
- Via dashboard: Render → Service → Manual Deploy → select commit
- Via API: `curl -X POST $RENDER_DEPLOY_HOOK`
- NEVER deploy on Fridays or before 3 pm ET

### Rollback
1. Render dashboard → Service → Deploys tab
2. Find last successful deploy
3. Click "Rollback to this deploy"
4. Verify: `curl https://staging.yourapp.com/health`

## Monitoring
- Error tracking: Sentry (`SENTRY_DSN` env var)
- Uptime: Better Uptime → alerts to `#oncall` Slack channel
- Logs: Render log streaming → Papertrail
- Performance: Render metrics dashboard

## Onboarding New Developers
1. Clone the repo
2. Copy `.env.example` to `.env`
3. `docker compose up -d` — start local services
4. `npm install`
5. `npx prisma migrate dev` — apply schema migrations
6. `npx prisma db seed` — seed development data
7. `npm run dev`
8. Open `http://localhost:5173`
9. Login: `dev@example.com` / `password123`

For access requests, post in `#dev-access` Slack.
