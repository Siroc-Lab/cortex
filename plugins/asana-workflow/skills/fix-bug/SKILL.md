---
name: fix-bug
version: 0.1.0
description: >
  Handles the full bug-fix lifecycle when start-task routes a Bug category Asana task.
  Orchestrates root cause investigation (superpowers:systematic-debugging), a TDD hard gate
  to prevent recurrence (superpowers:test-driven-development), and shipping (ship-it).
  Always called from start-task with full task context; never invoked directly.
---

# Fix Bug

Thin orchestrator for the bug-fix lifecycle. Contains no debugging or testing logic —
sequences three sub-skills with explicit gates. Always called from start-task, never directly.

## Inputs

Receives full task context from start-task:
- Task name, notes, custom fields (category, task ID, sprint)
- Subtasks (incomplete = remaining work, completed = done)
- Comments and attachments
- Branch name (already created and checked out)
- Asana task URL

## Step 1: Root Cause Investigation

Invoke `superpowers:systematic-debugging` with the full task context as the bug report.

Do not proceed to Step 2 until systematic-debugging confirms both:
- Root cause identified with specificity (not "it was broken")
- Fix implemented on the current branch

## Step 2: TDD Hard Gate

**This step cannot be skipped under any circumstance.**

After systematic-debugging declares the fix ready, invoke `superpowers:test-driven-development`
with these explicit requirements:

1. **Write a regression test** that:
   - Would fail if the fix were reverted
   - Has a name that makes the bug recognizable (e.g., `test_session_expiry_uses_utc_not_local`)

2. **Run the full test suite.** All three must be true before proceeding:
   - The new regression test passes
   - No previously-passing tests are now failing
   - Any pre-existing failures are documented as pre-existing

**If the gate cannot be satisfied** (tests stay red, no meaningful test can be written):
- Do NOT proceed to Step 3
- Return to `superpowers:systematic-debugging` Phase 1 — the fix is not considered complete
- If the gate still cannot be satisfied after a second pass, halt and surface the blocker
  to the user with a summary of what was attempted

## Step 3: Ship

Invoke `ship-it` with the following context:
- Summary of root cause and fix (from systematic-debugging)
- Regression test name and file path
- Test run output confirming all green

`ship-it` handles PR creation and Asana task update.
