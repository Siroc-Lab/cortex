---
name: pre-ship-check
metadata:
  version: 0.1.0
description: >
  This skill should be used when the user says "am I ready to ship", "pre-flight check",
  "check before PR", "ready to merge", "pre-ship check", "can I ship this", "is this ready",
  or "run checks". Validates that code is in a shippable state by running git-check for branch
  and working tree validation, then verifying lint, build, and tests pass. Works standalone
  or as the first step in the ship-it orchestrator.
---

# Pre-Ship Check

Readiness gate that validates code is in a shippable state. Combines git state validation (via `git-check`) with project-level build verification.

## Usage Modes

- **Standalone** — The user asks "am I ready to ship?" and wants a full status report.
- **Orchestrator step** — Called by `ship-it` as its first step. Blocking findings halt the pipeline.

## Step 1: Git State Validation

Invoke `git-check`. If it returns blocking issues, stop and resolve them before continuing. Advisory warnings are presented to the user to decide whether to proceed.

## Step 2: Run Project Commands

Ask the user for the test, build, and lint commands to run if they are not already known from context. Any command can be skipped by the user.

Run in this order (cheapest first):

| # | Command | Severity | Notes |
|---|---|---|---|
| 1 | Lint (declared `lint:` command) | ADVISORY | Warn but don't block |
| 2 | Build (declared `build:` command) | BLOCKING | Stop if fails |
| 3 | Tests (declared `test:` command) | BLOCKING | Ask before running (may be slow) |

Before running the test suite, ask:

> The test suite may take a while. Run it now, or skip and mark as unchecked?

## Output Format

Combine findings from git-check and project commands into a single report:

```
BLOCKING
  - Build failing: exit code 1

WARNINGS
  - Lint errors on changed files
  - Debug artifact: console.log found in src/utils/helper.ts (line 42)

PASSED
  - Git checks passed
  - Tests passing
```

Omit any section that has no findings. If everything passes:

```
PASSED
  - All checks passed. Ready to ship.
```

## Behavior Rules

1. **Blocking findings** — Stop and report. The user can override with explicit confirmation ("yes, ship anyway"), but make the risk clear.

2. **Advisory warnings only** — List them and ask: "These are non-blocking warnings. Continue anyway?"

3. **All checks pass** — Say so and proceed. No unnecessary friction.

4. **Long-running commands** — Ask before running test suites or builds that may take more than a few seconds. Let the user skip if they've already run them recently.

5. **Error handling** — Never silently skip a check. If a command fails unexpectedly (e.g., command not found), report the failure explicitly and note that the check could not be completed.
