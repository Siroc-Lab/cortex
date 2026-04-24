# Frontend Testing Infrastructure

Runner-specific coverage configs, CI pipeline patterns, and tooling for frontend projects. Extends `../../generic-testing/references/infrastructure.md` for universal infrastructure principles.

## Coverage Setup

### Jest

```json
{
  "coverageProvider": "v8",
  "collectCoverageFrom": [
    "src/**/*.{ts,tsx}",
    "!src/**/*.d.ts",
    "!src/**/index.{ts,tsx}",
    "!src/**/*.stories.{ts,tsx}"
  ],
  "coverageThreshold": {
    "global": {
      "branches": 70,
      "functions": 70,
      "lines": 70,
      "statements": 70
    }
  }
}
```

### Vitest

```typescript
export default defineConfig({
  test: {
    coverage: {
      provider: 'v8',
      include: ['src/**/*.{ts,tsx}'],
      exclude: ['**/*.d.ts', '**/index.{ts,tsx}', '**/*.stories.{ts,tsx}'],
      thresholds: {
        branches: 70,
        functions: 70,
        lines: 70,
        statements: 70
      }
    }
  }
})
```

### Notes

- **v8 provider** preferred over babel/istanbul — faster, more accurate for modern code
- **Exclude:** type definitions, barrel exports, stories, generated code, config files
- Adjust thresholds per project maturity — 70% is a starting point

## CI Pipeline (Frontend)

### Minimum viable test job

```yaml
# Pseudo — adapt to project's CI system
steps:
  - Install dependencies (cached by lockfile hash)
  - Type check (tsc --noEmit)
  - Lint (eslint / biome)
  - Unit + integration tests with coverage
  - Upload coverage report
```

Fail on any step. No `continue-on-error` for tests.

### E2E job (separate)

E2E needs browser binaries (Playwright/Cypress) and possibly a running dev server or test backend. Run as a separate CI job:

- Install browser binaries (cached)
- Start dev server or use a test environment
- Run E2E suite
- Upload traces/screenshots on failure

### PR gates

| Gate | Required | Advisory |
|---|---|---|
| Type check (`tsc`) | Yes | — |
| Lint | Yes | — |
| Unit/integration tests | Yes | — |
| Coverage threshold | Yes | — |
| Coverage delta | — | Yes (flag decreases) |
| E2E tests | Depends on project | — |
| Build succeeds | Yes | — |
