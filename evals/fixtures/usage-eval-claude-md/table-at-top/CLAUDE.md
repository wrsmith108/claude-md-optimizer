# Project Instructions

## Sub-Documentation

| Document | Description |
|----------|-------------|
| [Testing Guide](../docs/testing.md) | Test framework, file naming, coverage threshold |
| [Docker Guide](../docs/docker.md) | Container setup, service ports, troubleshooting |
| [CI Reference](../docs/ci.md) | Pipeline steps, branch protection, secrets |
| [Architecture](../docs/architecture.md) | Service layout, domain boundaries, design decisions |
| [Security Rules](../docs/security.md) | Auth, secrets, rate limiting, input validation |

## Build Commands

```bash
npm run build          # compile TypeScript
npm run dev            # start dev server with hot reload
npm test               # run unit tests
npm run test:coverage  # run tests with coverage report
npm run lint           # ESLint + Prettier check
npm run typecheck      # tsc --noEmit
```

## PR Conventions

- Branch from `main`, name branches `feat/`, `fix/`, `chore/`
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `chore:`)
- Keep PRs under 400 lines of diff; split larger changes

## Code Style

- TypeScript strict mode enabled — no `any`, no `@ts-ignore`
- Prefer `const` over `let`; avoid `var`
- Use absolute imports from `src/` (configured in `tsconfig.json`)
