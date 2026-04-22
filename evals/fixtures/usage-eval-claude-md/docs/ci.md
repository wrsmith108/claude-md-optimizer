# CI Reference

## Pipeline Overview

CI runs on every PR via GitHub Actions (`.github/workflows/ci.yml`). Total timeout: **22 minutes**.

## Required Checks

All checks must pass before merge:
1. `lint` — ESLint + Prettier (2 min)
2. `typecheck` — `tsc --noEmit` (3 min)
3. `test` — Vitest unit tests (8 min)
4. `integration` — Vitest integration tests against real DB (9 min)

## Branch Protection

- Requires **3 approving reviews** before merge
- Stale reviews dismissed on new commits
- Linear history enforced (no merge commits)

## Secrets in CI

Secrets are injected via GitHub Actions environment. Never hardcode credentials. Use `process.env.STRIPE_SECRET_KEY` pattern.

## Flaky Tests

Mark known-flaky tests with `it.skip` and open a tracking issue. Do not retry flaky tests in CI — fix them.
