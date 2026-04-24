# Frontend Testing

Frontend-specific testing patterns, tooling, and infrastructure. Extends the universal fundamentals in `../generic-testing/process.md`.

## Base Fundamentals

Read and follow `../generic-testing/process.md` for the non-negotiables. This document provides the frontend bindings.

## The Golden Rule

> Query the way a user would find the element. Interact the way a user would interact.

No `querySelector`. No internal state inspection. No `wrapper.instance()`. If a user can't see it or do it, don't test it.

## Stack Detection

Before writing any test, detect the project's stack. See `references/stack-detection.md`.

## Query Priority

Testing Library provides queries in priority order. Use the highest-priority query that works.

| Priority | Query | Use when |
|---|---|---|
| **1. Accessible** | `getByRole`, `getByLabelText` | Always try first. This is how assistive tech finds elements. |
| **2. Semantic** | `getByText`, `getByDisplayValue` | Visible text content |
| **3. Test ID** | `getByTestId` | Last resort. Element has no accessible name or visible text. |

**Never use:** `container.querySelector`, class names, or internal component references.

**Why this order matters:** If your test can't find the element by role or label, your UI might have an accessibility problem. The test is telling you something.

## Component Tests

### Render and Assert

Test what the user sees after the component renders.

```
Arrange — render the component with props/context
Act     — (none for render tests)
Assert  — query for visible content
```

**Test real output.** Don't assert on component internals, state variables, or re-render counts.

### User Interactions

Simulate real user behavior — clicks, typing, keyboard navigation.

```
Arrange — render the component
Act     — simulate user interaction (click, type, press key)
Assert  — verify the visible result of the interaction
```

**Use `userEvent` over `fireEvent`.** `userEvent` simulates real browser behavior (focus, blur, keydown, keyup, input). `fireEvent` dispatches a single synthetic event. The gap between them is where bugs hide.

### Async Behavior

For components that fetch data, show loading states, or update after async operations.

```
Arrange — render with mocked data source (MSW, stub, or fixture)
Act     — trigger the async operation
Assert  — wait for the result using `findBy*` or `waitFor`
```

**Never use fake timers to skip loading states.** If the loading state exists, test it. Users see it.

**Never assert on intermediate renders.** Assert on the final visible state.

## What to Mock

### Mock at the network boundary, not the component boundary.

| Layer | Mock? | Why |
|---|---|---|
| **HTTP requests** | Yes — MSW or equivalent | Deterministic, no real server needed |
| **Browser APIs** (localStorage, geolocation, clipboard) | Yes — stub or fake | Not available in test environment |
| **Third-party SDKs** (analytics, auth providers) | Yes — stub the SDK | Avoid side effects, control responses |
| **Child components** | Almost never | You lose integration confidence |
| **Hooks/utilities** | Almost never | Test through the component that uses them |
| **State management** | Almost never | Render with real store, seed with test data |

**MSW (Mock Service Worker)** is the preferred way to mock network requests. It intercepts at the network level, so your code runs exactly as it would in production — fetch calls, error handling, headers, all real. If the project doesn't have MSW, stub fetch/axios at the module level as a fallback.

### Router and Navigation

Don't mock the router. Wrap in a test router with an initial route.

### Auth and Providers

Don't mock context providers. Render with real providers, seed with test state. If the provider has side effects (network calls), mock those at the network boundary.

## Hooks

Test hooks through the components that use them. If a hook is complex enough to test in isolation, use `renderHook` — but prefer component tests.

**Don't test implementation details of hooks.** Test that the component using the hook behaves correctly.

## Forms

Test the full user flow: fill fields, submit, verify outcome.

- Use `userEvent.type()` not `fireEvent.change()`
- Test validation messages appear for invalid input
- Test submit with valid data produces the right outcome
- Test that disabled/loading states prevent double submission

## Accessibility in Tests

Testing Library's query priority naturally tests accessibility. If you can't find an element by role, your UI has an accessibility gap.

Additionally:
- Verify focus management after interactions (modals, menus, form errors)
- Test keyboard navigation for interactive elements
- Check that error messages are associated with their inputs (`aria-describedby`)

## E2E Tests (Playwright/Cypress)

E2E tests cover critical user flows through the full stack. They complement component tests — they don't replace them.

### What to E2E test

- **Critical paths only:** sign up, sign in, core feature happy path, checkout
- **Flows that cross multiple pages/views**
- **Flows that depend on real backend behavior** (auth, payments, permissions)

### What NOT to E2E test

- Individual component states (use component tests)
- Every form validation message (use component tests)
- Edge cases in business logic (use integration tests)

### E2E Principles

- **No shared state between tests.** Each test seeds its own data.
- **No `sleep` or `waitForTimeout`.** Use auto-waiting or explicit `waitFor` conditions.
- **Selectors:** prefer `getByRole`, `getByText`, `getByTestId` (same priority as component tests). Avoid CSS selectors.
- **One flow per test.** Don't chain unrelated assertions.

## Snapshot Tests

Use sparingly. Targeted assertions on specific values are almost always better.

**Acceptable uses:**
- Serialized data structures (API response shapes, config objects)
- Generated output that's hard to assert on field by field

**Never use for:**
- Component render output (too brittle, diffs are unreadable)
- Large objects where you only care about a few fields
- Anything where the snapshot is updated without reading the diff

## Infrastructure

See `references/infrastructure.md` for frontend-specific coverage configs, CI pipeline templates, and tooling setup.
