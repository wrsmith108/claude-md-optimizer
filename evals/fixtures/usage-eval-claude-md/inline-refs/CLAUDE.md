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

For test file naming conventions, the coverage threshold, and framework details, see [docs/testing.md](../docs/testing.md).

## Docker

For service ports and container troubleshooting, see [docs/docker.md](../docs/docker.md).

## CI Pipeline

For required checks, branch protection rules, and secret injection, see [docs/ci.md](../docs/ci.md).

## Architecture

For service layout, domain boundaries, and key design decisions, see [docs/architecture.md](../docs/architecture.md).

## Security

For auth patterns, password hashing, rate limiting, and input validation, see [docs/security.md](../docs/security.md).

## PR Conventions

- Branch from `main`, name branches `feat/`, `fix/`, `chore/`
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `chore:`)
- Keep PRs under 400 lines of diff; split larger changes

## Code Style

- TypeScript strict mode enabled — no `any`, no `@ts-ignore`
- Prefer `const` over `let`; avoid `var`
- Use absolute imports from `src/` (configured in `tsconfig.json`)
