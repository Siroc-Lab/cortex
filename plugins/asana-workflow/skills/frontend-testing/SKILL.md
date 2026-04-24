---
name: frontend-testing
version: 0.1.0
description: >
  Use when writing, reviewing, or improving tests in a frontend project — triggers include "write tests",
  "add tests", "test this component", "improve test coverage", "/frontend-testing", or any request to create
  or fix frontend tests. Also triggered when TDD is active in a frontend codebase to provide the testing
  context and patterns. Works with any frontend framework (React, SolidJS, Vue, Svelte) and test runner
  (Jest, Vitest, Playwright, Cypress).
---

# Frontend Testing

Write frontend tests that prove behavior, resist refactors, and enforce quality through infrastructure.

## Base Fundamentals

Read and follow `../generic-testing/process.md` — the 10 non-negotiables apply to every test you write.

## Process

Read and follow `process.md` for frontend-specific patterns: component testing, query priority, mocking boundaries, E2E, and accessibility.

## Before Writing Any Test

1. **Detect the stack** — read `references/stack-detection.md` and run the detection steps. Know the runner, framework, test utils, coverage tool, and file conventions before writing a single line.
2. **Match existing conventions** — file naming, import style, test structure. Don't introduce a second pattern.

## Reference Files

- **`../generic-testing/process.md`** — Universal testing fundamentals (determinism, AAA, behavior over implementation)
- **`../generic-testing/references/infrastructure.md`** — CI, flake detection, benchmarks, reporting (stack-agnostic)
- **`process.md`** — Frontend testing patterns (Testing Library, components, hooks, forms, E2E)
- **`references/stack-detection.md`** — Detect frontend runner, framework, coverage, package manager
- **`references/infrastructure.md`** — Jest/Vitest coverage configs, frontend CI pipeline
