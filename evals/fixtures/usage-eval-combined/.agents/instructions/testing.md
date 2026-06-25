---
description: "Test framework, file naming convention (.unit.spec.ts), coverage threshold (87% for src/services/ and src/api/), and mock patterns using vi.mock()"
loading_strategy: "lazy"
---

# Testing Guide

## Framework

All tests use **Vitest** (not Jest). Do not add Jest configuration.

## File Naming Convention

Test files must be co-located with the source file and use the `.unit.spec.ts` suffix:

```
src/services/payments/stripe-processor.ts
src/services/payments/stripe-processor.unit.spec.ts
```

Integration tests use `.integration.spec.ts`. End-to-end tests live in `e2e/` and use `.e2e.spec.ts`.

## Coverage Threshold

Minimum **87% line coverage** is enforced in CI for all files under `src/services/` and `src/api/`. Coverage below this blocks merge.

Run coverage locally:
```bash
npx vitest run --coverage
```

## Test Structure

Use `describe` / `it` blocks. Mock external dependencies with `vi.mock()`. Avoid `any` type assertions in test files.

## Snapshot Testing

Snapshots are stored in `__snapshots__/` alongside the test file. Update with `npx vitest --update-snapshots`.
