# Frontend Stack Detection

How to detect the testing stack in a frontend project. Run these checks **before** writing any test.

## Detection Order

### 1. Test Runner

Check config files in project root:

| File found | Runner |
|---|---|
| `vitest.config.*` or `vite.config.*` with `test` block | Vitest |
| `jest.config.*` or `"jest"` in package.json | Jest |
| `playwright.config.*` | Playwright (E2E) |
| `cypress.config.*` | Cypress (E2E) |

If both Jest and Vitest configs exist, check which one `package.json` scripts reference.

### 2. Test Utilities

Check `package.json` dependencies:

| Package | Provides |
|---|---|
| `@testing-library/react` | React component testing |
| `@solidjs/testing-library` or `solid-testing-library` | SolidJS component testing |
| `@testing-library/vue` | Vue component testing |
| `@testing-library/svelte` | Svelte component testing |
| `@testing-library/dom` | DOM queries (framework-agnostic) |
| `@testing-library/user-event` | User interaction simulation |
| `@testing-library/jest-dom` | DOM-specific matchers (toBeInTheDocument, etc.) |
| `msw` | Network request mocking (Mock Service Worker) |

### 3. Framework

Check `package.json` dependencies:

| Package | Framework |
|---|---|
| `next` | Next.js (React) |
| `react`, `react-dom` | React |
| `solid-js` | SolidJS |
| `vue` | Vue |
| `svelte` | Svelte |

### 4. Coverage

Check test runner config for coverage settings:

| Config | Tool |
|---|---|
| Jest `coverageProvider: "v8"` | v8 via Jest |
| Jest `coverageProvider: "babel"` | Istanbul via Jest |
| Vitest `coverage.provider: "v8"` | v8 via Vitest |
| Vitest `coverage.provider: "istanbul"` | Istanbul via Vitest |
| `nyc` or `c8` in package.json | Standalone coverage |

### 5. Package Manager

| File found | Manager | Run command |
|---|---|---|
| `pnpm-lock.yaml` | pnpm | `pnpm test` |
| `yarn.lock` | yarn | `yarn test` |
| `package-lock.json` | npm | `npm test` |
| `bun.lockb` | bun | `bun test` |

### 6. CI Pipeline

Check `.github/workflows/` for test jobs. Note what's already running (lint, test, build, E2E) and what's missing.

### 7. Test File Conventions

Check existing test files for patterns:

| Pattern | Convention |
|---|---|
| `*.test.ts` / `*.test.tsx` | Co-located test files |
| `*.spec.ts` / `*.spec.tsx` | Co-located spec files |
| `__tests__/` directory | Grouped test directory |

Match whatever the project already uses. Don't introduce a second convention.

## Output

After detection, you should know:
- **Runner:** Jest / Vitest / other
- **Test utils:** Which @testing-library packages
- **Framework:** React / Solid / Vue / Svelte
- **Coverage:** Configured or not, which provider
- **Package manager:** npm / yarn / pnpm / bun
- **CI:** What runs, what doesn't
- **Test file pattern:** `*.test.*`, `*.spec.*`, or `__tests__/`

Use this to adapt all test code to match the project's existing conventions.
