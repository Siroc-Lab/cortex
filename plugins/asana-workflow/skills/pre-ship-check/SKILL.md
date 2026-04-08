---
name: pre-ship-check
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

## Step 1: QA Verification Gate (Bug Tasks)

**Only applies when a task GID is in context and the task category is Bug.**

Before any other checks, verify that QA verification has passed by checking the Asana task's comments (via `asana-api` Fetch Task Stories) for a comment containing `✅ QA Verification — PASSED`.

- **Found** → QA gate passes. Proceed to Step 2.
- **Not found** → **BLOCKING**. Report:
  > QA verification has not passed for this bug task. The fix must be verified via the QA skill before shipping.
  > Run the verify step (start-task Step 10d or fix-bug Step 3) to generate the verification.

This gate cannot be overridden. A bug fix without runtime verification evidence is not shippable.

**Skip when:** task is not a Bug category, or no task GID is in context.

## Step 2: Git State Validation

Invoke `git-check`. If it returns blocking issues, stop and resolve them before continuing. Advisory warnings are presented to the user to decide whether to proceed.

## Step 3: Resolve Project Commands

Scan CI pipelines to infer test, build, and lint commands (see below), then confirm with the user before running.

### CI Pipeline Inference

When commands are missing, scan for CI pipeline files in this order:
1. `.github/workflows/*.yml` / `.yaml` — look for steps with `run:` containing test/build/lint keywords
2. `.circleci/config.yml` — look for job steps
3. `Makefile` — look for targets named `test`, `build`, `lint`, `check`
4. `package.json` `scripts` — look for keys matching `test`, `build`, `lint`
5. `Taskfile.yml` / `justfile` — look for matching tasks

Extract the most specific matching command for each of `test`, `build`, `lint`. Then present inferred commands to the user for confirmation before running:

> I found these commands from your CI pipeline — confirm or edit before I run them:
> - lint: `yarn lint`
> - build: `yarn build`
> - test: `yarn test`
>
> Proceed with these? (You can skip any.)

If no commands can be inferred for a given check after scanning all sources, mark that check as **SKIPPED (not configured)** — not blocking.

## Step 4: Run Project Commands

Run in this order (cheapest first):

| # | Command | Severity | Notes |
|---|---|---|---|
| 1 | Lint (declared or inferred `lint` command) | ADVISORY | Warn but don't block |
| 2 | Build (declared or inferred `build` command) | BLOCKING | Stop if fails |
| 3 | Tests (declared or inferred `test` command) | BLOCKING | Ask before running (may be slow) |

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
