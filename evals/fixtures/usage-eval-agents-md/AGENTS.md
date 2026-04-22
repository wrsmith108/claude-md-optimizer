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

For test file naming conventions, coverage threshold, and framework details, see [.agents/instructions/testing.md](.agents/instructions/testing.md).

## Docker

For local setup and service ports, see [.agents/instructions/docker.md](.agents/instructions/docker.md).

## CI Pipeline

For required checks and branch protection rules, see [.agents/instructions/ci.md](.agents/instructions/ci.md).

## Architecture

For service layout, domain boundaries, and key design decisions, see [.agents/instructions/architecture.md](.agents/instructions/architecture.md).

## Security

For auth patterns, secrets management, and input validation, see [.agents/instructions/security.md](.agents/instructions/security.md).

## PR Conventions

- Branch from `main`, name branches `feat/`, `fix/`, `chore/`
- Commit messages follow Conventional Commits (`feat:`, `fix:`, `chore:`)
- Keep PRs under 400 lines of diff; split larger changes

## Code Style

- TypeScript strict mode enabled — no `any`, no `@ts-ignore`
- Prefer `const` over `let`; avoid `var`
- Use absolute imports from `src/` (configured in `tsconfig.json`)
