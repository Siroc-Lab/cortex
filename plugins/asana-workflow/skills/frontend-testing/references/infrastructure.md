# Frontend Testing Infrastructure

Frontend-specific coverage and CI guidance. Extends `../../generic-testing/references/infrastructure.md` for the universal principles.

Before applying any of the snippets below, read the project's existing runner config and CI workflows. Adapt — don't paste.

## Coverage

### Principles

- **Provider:** prefer the runner's V8 provider over Babel/Istanbul instrumentation — it's faster and more accurate for modern bundled code. Use whatever the runner already configures unless you have a reason to switch.
- **Include paths:** mirror the project's actual source layout (`src/`, `app/`, `packages/*/src`, `lib/`, etc.). Inspect the repo, don't assume.
- **Exclude:** type declarations (`*.d.ts`), barrel files (`index.*` re-exports), stories, generated code, config files, and the test files themselves.
- **Thresholds:** start around 70% on a greenfield project. On an existing codebase, set thresholds at the current level and ratchet up — never set thresholds that fail the build on day one.

### Where to put thresholds

Most runners support per-directory overrides. Push higher thresholds (90%+) on critical paths (auth, payments, data integrity) and accept lower ones on UI surface or glue code. A single global number hides where the suite is weak.

## CI Pipeline (Frontend)

### Minimum viable test job

```yaml
# Pseudo — adapt to the project's CI system and package manager
steps:
  - Install dependencies (cached by lockfile hash)
  - Type check
  - Lint
  - Unit + integration tests with coverage
  - Upload coverage report
```

Fail on any step. No `continue-on-error` for tests.

### E2E job (separate)

E2E typically needs browser binaries and possibly a running dev server or test backend. Run it as a separate job:

- Install browser binaries (cached)
- Start the dev server or point at a test environment
- Run the E2E suite
- Upload traces/screenshots on failure

### PR gates

| Gate | Required | Advisory |
|---|---|---|
| Type check | Yes | — |
| Lint | Yes | — |
| Unit/integration tests | Yes | — |
| Coverage threshold | Yes | — |
| Coverage delta | — | Yes (flag decreases) |
| E2E tests | Depends on project | — |
| Build succeeds | Yes | — |
