# Copilot Instructions — FullStack App

## Project Overview

Full-stack web application with a React/TypeScript frontend and a Python/FastAPI backend. Monorepo layout: `frontend/` and `backend/` are top-level directories. Shared OpenAPI spec lives in `api/openapi.yaml`; the frontend client is auto-generated from it.

## Quick Commands

```bash
# Backend
cd backend && uvicorn app.main:app --reload   # dev server (port 8000)
cd backend && pytest                           # run tests
cd backend && pytest --cov=app --cov-report=html  # tests + coverage
cd backend && ruff check . && ruff format .    # lint + format
cd backend && mypy app/                        # type-check

# Frontend
cd frontend && npm run dev                     # dev server (port 5173)
cd frontend && npm test                        # run Vitest unit tests
cd frontend && npm run test:e2e                # Playwright e2e tests
cd frontend && npm run build                   # production build
cd frontend && npm run typecheck               # tsc --noEmit
cd frontend && npm run lint                    # ESLint

# API client codegen (run from repo root)
npm run codegen                                # regenerates frontend/src/api/client.ts from openapi.yaml
```

## Security
- Never commit secrets, API keys, or credentials to the repository
- Backend validates all user input via Pydantic models — never trust raw request dicts
- Use parameterized SQLAlchemy queries — never interpolate user input into SQL
- Frontend: sanitize any user-generated HTML before rendering (`DOMPurify`)
- CORS is restricted to `ALLOWED_ORIGINS` env var — do not hardcode origins

## TypeScript Rules (Frontend)

### Type Safety
- Enable `"strict": true` in `tsconfig.json` — no exceptions
- No `any` types — use `unknown` or proper generics
- Avoid `!` non-null assertions — handle nullability with optional chaining or explicit checks
- Prefer `type` over `interface` for object shapes

### React Patterns
- Function components only — no class components
- Use custom hooks (`useXxx`) for shared stateful logic; keep components lean
- Co-locate test files: `MyComponent.tsx` + `MyComponent.test.tsx`
- CSS Modules for component styles (`MyComponent.module.css`)
- Named exports only — no default exports for components or hooks
- State management: React Query for server state, Zustand for UI state

### API Client
- Always use the auto-generated client in `frontend/src/api/client.ts` — never hand-write `fetch` calls to backend endpoints
- If the API contract changes, update `api/openapi.yaml` first, then run `npm run codegen`
- React Query hooks wrap the generated client; put them in `frontend/src/hooks/api/`

### Testing (Frontend)
- Unit tests: Vitest; co-located with source
- E2E tests: Playwright; in `frontend/tests/e2e/`
- Mock API calls using MSW (Mock Service Worker) — fixtures in `frontend/src/mocks/`
- Minimum 75% line coverage for `frontend/src/`
- Snapshots discouraged — prefer explicit assertions

## Python Rules (Backend)

### Type Safety
- Type annotations required on all function signatures
- Pydantic v2 models for all request/response schemas
- `mypy --strict` must pass — CI blocks on type errors

### FastAPI Patterns
- Async everywhere: `async def` for all route handlers and DB operations
- Dependency injection via `Depends()` for DB sessions, auth checks, rate limiters
- Route handlers are thin: validate input → call service → return response
- Business logic in `app/services/` — never in route handlers or repositories
- HTTP errors: raise `HTTPException` in route handlers; use domain exceptions in services

### Database (SQLAlchemy + Alembic)
- SQLAlchemy 2.0 async ORM — always use `async with AsyncSession` context manager
- Alembic for migrations: `alembic revision --autogenerate -m "description"`, `alembic upgrade head`
- NEVER use `session.execute(text("..."))` with user data — parameterize everything
- Models in `app/models/`, schemas in `app/schemas/`

### Testing (Backend)
- pytest + pytest-asyncio; fixtures in `tests/conftest.py`
- Use `httpx.AsyncClient` (not `requests`) for testing FastAPI endpoints
- Test DB: SQLite in-memory for unit tests; use `pytest-postgresql` for integration tests
- Coverage target: 85% for `app/`; CI fails below threshold

## PR & Commit Rules
- Conventional Commits: `feat(frontend): add dark mode toggle`, `fix(backend): handle null user in auth middleware`
- Tests required for new endpoints and non-trivial frontend components
- Run codegen (`npm run codegen`) if `api/openapi.yaml` changes and commit the updated client
- Draft PRs for work in progress; mark ready when CI is green
- 1 approving review required; squash-merge only

## Architecture Overview

### Request Flow
```
Browser → React (Vite)
  → React Query hook
  → Generated API client (frontend/src/api/client.ts)
  → FastAPI backend (port 8000)
  → Service layer (app/services/)
  → Repository layer (app/repositories/)
  → PostgreSQL (via SQLAlchemy async)
```

### Directory Structure
```
frontend/
  src/
    api/          # auto-generated client (do not edit by hand)
    components/   # UI components
    hooks/        # custom hooks (api/ for React Query, ui/ for UI state)
    pages/        # page-level components
    store/        # Zustand stores
    utils/        # pure utilities
backend/
  app/
    api/          # FastAPI route handlers
    services/     # business logic
    repositories/ # DB access via SQLAlchemy
    models/       # SQLAlchemy ORM models
    schemas/      # Pydantic v2 schemas
    middleware/   # FastAPI middleware
    config.py     # env-var-backed settings (pydantic-settings)
api/
  openapi.yaml    # source of truth for API contract
```

## CI/CD Pipeline

**ci-backend.yml** — runs on every PR affecting `backend/`:
1. `uv sync`
2. `ruff check . && ruff format --check .`
3. `mypy app/ --strict`
4. `pytest --cov=app --cov-report=xml`
5. Coverage < 85% fails pipeline

**ci-frontend.yml** — runs on every PR affecting `frontend/`:
1. `npm ci`
2. `npm run typecheck`
3. `npm run lint`
4. `npm test -- --coverage`
5. Coverage < 75% fails pipeline

**e2e.yml** — runs on push to `main`:
1. `docker compose up -d` (postgres, redis)
2. Build backend + frontend
3. Start both servers
4. `npm run test:e2e`
5. Upload Playwright report as artifact

**deploy.yml** — runs on tag `v*`:
1. Build Docker images for backend and frontend (nginx static)
2. Push to ECR
3. Deploy to ECS (Terraform)
4. Health check: `GET /health` must return 200 within 5 min

## Troubleshooting

### Codegen produces unexpected output
1. Validate `api/openapi.yaml` first: `npx @apidevtools/swagger-parser validate api/openapi.yaml`
2. Check the generator config in `codegen.config.json`
3. Stale codegen output: delete `frontend/src/api/client.ts` and re-run `npm run codegen`

### mypy errors after pulling
Usually a new dependency added without stubs. Check `backend/pyproject.toml` for `mypy` section; add `types-<package>` stub package or `[[tool.mypy.overrides]]` ignore if stubs don't exist.

### "CORS policy" error in browser
1. Check `ALLOWED_ORIGINS` in backend `.env` includes `http://localhost:5173`
2. Verify the backend is running and `OPTIONS` preflight returns 200
3. Check that you're not hitting the prod URL from local dev

### Playwright tests timing out
- Default timeout is 30 s. Increase: `PLAYWRIGHT_TIMEOUT=60000 npm run test:e2e`
- Screenshots captured on failure: look in `frontend/test-results/`
- Check that both backend and frontend are running before tests start

### React Query cache stale
For dev debugging: `queryClient.invalidateQueries()` in the browser console, or wrap component in `<ReactQueryDevtools />` (already configured in `App.tsx` for dev mode).
