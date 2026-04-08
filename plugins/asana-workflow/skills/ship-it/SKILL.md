---
name: ship-it
version: 0.1.0
description: >
  This skill should be used when development is complete and the work needs to be shipped:
  the user says "ship it", "we're done", "ready to ship", "done with this feature", "let's wrap up",
  "mark as in review", "create the PR", "link this to Asana", "create a PR and update Asana",
  or "push this and close the ticket". Also triggered automatically by `start-task` and
  `start-task-steps` once the downstream development workflow (feature-dev, fix-bug, brainstorm)
  signals completion — in that case all session context (task GID, branch, draft PR URL) is
  already available and must not be re-requested.
---

# Ship It

Thin orchestrator that calls sub-skills in sequence to ship completed work. This skill contains NO domain logic — it coordinates, threads context, and handles skip conditions. Each sub-skill is independently invocable and self-contained.

## Prerequisites

### Sub-skills (must be installed)

- `pre-ship-check` — readiness gate (clean tree, commits, branch state)
- `work-summary` — session recap (git history, conversation context)
- `create-pr` — PR lifecycle (create, push, format)
- `asana-api` — Asana API operations (fetch, comment, move sections)

### External tools

- `gh` CLI authenticated for GitHub
- `asana-api` skill for Asana API operations — handles token resolution and setup guidance.

## Context Threading

The orchestrator threads context from earlier in the session — it does NOT re-ask for information that is already available.

### From start-task (if used)

When the session began with `start-task`, the following context is already in the conversation:
- **Task GID** — extracted from the Asana URL
- **Task URL** — the original Asana task URL
- **Task ID** — the project ID (e.g., MT251-168) from the custom field
- **Sprint project GID** — from the task's memberships
- **Section mappings** — already discovered when moving to "In Progress"
- **Branch name** — created by start-task (e.g., `MT251-168/add-export`)
- **Draft PR URL** — created by start-task, to be promoted to ready by create-pr

Reuse all of this. Do not ask the user for the Asana task URL again.

### From conversation history

If `start-task` was not used but an Asana URL appeared earlier in the conversation, extract the task GID from it. Only prompt for the URL if there is genuinely no Asana context available.

## The Flow

Follow these 6 steps in order.

### Step 1: Pre-ship Check

Invoke `pre-ship-check`.

- If it returns **blocking** issues — stop and resolve them before continuing.
- If it returns **advisory** warnings (including QA verification) — present them to the user and ask whether to proceed or fix first.

### Step 2: QA Verification (if needed)

If pre-ship-check reported a QA advisory (no QA verification found for a non-bug task), offer to run QA before continuing:

> "No QA verification found. Would you like to run QA to verify the changes before shipping?
> This will build, deploy, and visually verify the affected flows. Evidence will be uploaded to the Asana task.
> - **Yes** — run QA verification
> - **Skip** — proceed without QA"

If the operator chooses **Yes**:
1. Resolve the QA skill: check CLAUDE.md for `qa-skill:` declaration, then infer from project files (same logic as start-task Step 10a).
2. Invoke the QA skill in **investigate** mode with a description of what was built/changed (from git diff/log).
3. The QA skill posts `✅ QA Verification — Feature Complete` to Asana with evidence.

If the operator chooses **Skip**, proceed to Step 3.

**Skip this step when:** pre-ship-check did not report a QA advisory (QA already done, or no task GID).

### Step 3: Work Summary

Invoke `work-summary` to generate a session recap.

Present the summary to the user and let them tweak it before proceeding. This summary will be reused in the PR description and Asana comment.

### Step 4: Create PR

Invoke `create-pr`, passing:
- The work summary from Step 3
- The Asana task URL (from context threading above, if available)
- `orchestrator: true` — signals create-pr to skip its own git-check (already done in Step 1)

If a draft PR exists from `start-task`, create-pr will promote it to ready, update its description with the work summary, and assign reviewers — no new PR needed.

### Step 5: Asana Update

Handle via the `asana-api` skill. All Asana operations use the task GID from context threading.

1. **Move to "In Review":** Find the Sprint project from the task's memberships, find the "In Review" section, and move the task there. If the section mappings are already in conversation context from start-task, reuse them — do not re-fetch.

2. **Post ship comment:** Post a comment on the task with the work summary and PR link. The comment MUST include the stats line from work-summary (`~Xm | Files changed: N | Commits: N`):

   ```
   <work summary body>

   ~Xm | Files changed: N | Commits: N

   PR: <pr-url>

   🤖 Done
   ```

   The stats line is mandatory. The `🤖 Done` footer is mandatory — it signals AI-assisted work is complete and ready for review.

**Skip condition:** If no Asana task context is available, skip this step entirely.

### Step 6: Recap

Print a single recap:

> Shipped! Here's what happened:
> - Pre-ship check: passed (or "blocked — resolved X")
> - Work summary: generated
> - PR created: <pr-url> (or "already existed" / "skipped")
> - Asana task moved to "In Review": <task-url> (or "skipped")
> - Asana comment posted (or "skipped")

## Skippable Steps Summary

| Condition | Steps skipped |
|---|---|
| No Asana task context | 2, 5 |
| QA already done or no advisory | 2 |
| Draft PR from start-task | 4 promotes draft to ready (no skip) |

## Deliberate Removals

- **No estimated cost** in session stats — removed from work-summary output.

## Error Handling

Never silently skip a step. If a sub-skill or command fails:
1. Report exactly what failed and why.
2. Ask the user how to proceed (retry, skip, or abort).
