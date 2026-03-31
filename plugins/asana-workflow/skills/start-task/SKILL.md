---
name: start-task
version: 0.1.0
description: >
  This skill should be used when the user provides an Asana task URL and wants to begin working on it —
  "start task", "work on this", "pick up this ticket", "let's start on [task-id]", "jump on this",
  "begin this task", or pastes an Asana URL with intent to start development. Orchestrates the full
  lifecycle: validates sprint-readiness, sets up the branch and draft PR, routes to the right
  development workflow (feature-dev, fix-bug, or brainstorm), and ships via ship-it when done.
  Also handles pausing blocked work ("park this", "I'm blocked", "pause task", "put this on hold")
  and resuming ("resume task", "pick up where I left off", "continue [task-id]"). Use this variant
  for straightforward tasks; use start-task-steps for long or complex tasks that need step-by-step
  checkpoint tracking.
argument-hint: <asana-task-url> [brainstorm|feature-dev]
---

# Start Task

Take an Asana task, validate it's ready for development, understand the work, set up the branch, and route to the right skill. This skill is the conductor — it validates, prepares, and hands off.

## Prerequisites

- `$ASANA_PERSONAL_ACCESS_TOKEN` env var set in `~/.zshrc` — the Asana personal access token. If missing, stop and guide setup:
  > Add to `~/.zshrc`: `export ASANA_PERSONAL_ACCESS_TOKEN="your-asana-token-here"`
  > Get token from: https://app.asana.com/0/my-apps
- Access to `feature-dev:feature-dev`, `superpowers:systematic-debugging`, `web-qa`, and optionally `superpowers:brainstorming` skills
- The `asana-api` skill for all Asana API operations

## The Flow

### Step 0: Check External Skill Dependencies

Before doing anything else, check whether the external skills required for routing are installed. These are **not bundled** with this plugin.

- **`feature-dev@claude-plugins-official`** — required for non-bug tasks using the `feature-dev` workflow
- **`superpowers@claude-plugins-official`** — required for Bug tasks (`fix-bug` uses `systematic-debugging`) and for non-bug tasks using the `brainstorm` workflow

If either is missing, warn the user and ask whether to install now or continue. This is an **advisory blocking step** — wait for the user's answer before proceeding. See **`references/skill-dependencies.md`** for check instructions, install commands, and warning message templates.

### Step 1: Get the Asana Task URL

The Asana task URL is passed as `$ARGUMENTS`. If empty or invalid, prompt for it. Extract the **task GID** from the URL. See **`references/asana-patterns.md`** for supported URL formats.

### Step 2: Fetch Task Details

Fetch the full task with custom fields, memberships, assignee, and notes via the `asana-api` skill. See **`references/asana-patterns.md`** for required `opt_fields`.

Present a quick summary for confirmation: task name, assignee, category, task ID, and sprint.

### Step 3: Validate Sprint-Readiness

Run four validation checks: Sprint project membership, Estimated time, Product Status = Assigned, and ID field presence. See **`references/validation-rules.md`** for check details, failure display format, fix-offer logic, and skip rules.

Report failures as a checklist. Sprint membership, Estimated time, and Product Status are all blocking — offer to set the latter two via API, but do not proceed until all three pass. Only the ID field can be skipped after a warning.

### Step 4: Fetch Subtasks

Fetch subtasks via the `asana-api` skill. Group by status (incomplete = remaining work, completed = already done). Include subtasks in downstream context so the receiving skill understands what "done" looks like.

### Step 5: Fetch Comments and Attachments

Fetch task stories and filter for comments. List attachments by name, noting any images (mockups, screenshots). See **`references/asana-patterns.md`** for details.

### Step 6: Check for Existing Work

Before creating a branch, check if work already exists for this task ID. See **`references/git-workflow.md`** for the detection commands.

If a branch or PR exists, offer to resume or start fresh. If resuming, check out the existing branch and skip creation.

### Step 6a: Check for Checkpoint (Resume)

After fetching remote refs, check for `.claude/checkpoints/<task-id>.md`. If found, this is a resume — present the checkpoint state, check for new Asana comments since the pause, and offer to resume. See **`references/checkpoints.md`** for the full resume flow and edge cases (deleted branch, completed task, no answer yet).

On resume, skip validation and branch creation — check out the existing branch and route directly to the workflow specified in the checkpoint. The same Step 11 ship-it handoff applies after the workflow completes.

### Step 6b: Ask About Worktree (BLOCKING)

Before creating the branch, ask the user whether to use a git worktree. This is a **blocking** question — wait for an explicit answer before proceeding.

Present the choice:

> Would you like to work in a git worktree (isolated copy of the repo) or directly in the current directory?
> - **Worktree** _(recommended for parallel work — keeps main directory clean)_
> - **Current directory**

If the user chooses worktree, use `EnterWorktree` to create an isolated copy. The branch will be created inside the worktree in Step 7.

### Step 6c: Confirm Base Branch (BLOCKING)

Ask the user which branch to base the new branch on. This is a **blocking** question — wait for an explicit answer before proceeding.

Present the choice:

> Which branch should `<task-id>/<slug>` be based on?
> - **main** _(default — latest stable base)_
> - Another branch _(enter branch name)_

Default to `main` only after the user confirms. If the user specifies a different base branch, use that instead. Record the chosen base branch for Step 7.

### Step 7: Create Feature Branch

Create a branch using the task ID and a slug from the task name. Use the **base branch confirmed in Step 6c** (not assumed `main`). Inform (do not ask) when creating. See **`references/git-workflow.md`** for commands and naming convention.

### Step 8: Create Draft PR

Immediately after creating the branch, create an empty commit and a draft PR to establish the GitHub ↔ Asana link from minute one.

1. Create an empty commit so the branch can be pushed:
   ```bash
   git commit --allow-empty -m "<task-id> :: <task-name-slug> (start)"
   ```

2. Push the branch:
   ```bash
   git push -u origin $(git branch --show-current)
   ```

3. Create the draft PR:
   ```bash
   gh pr create --draft --title "<TASK-ID> :: <description>" --body "$(cat <<'EOF'
   ## Asana Task
   <asana-task-url>
   EOF
   )"
   ```

4. Capture the draft PR URL — it will be included in the Asana start comment and threaded through to ship-it.

### Step 9: Move to In Progress + Post Comment

These happen automatically — no permission needed.

**Move the task** to "In Progress" on the Sprint board. Skip if already there. See **`references/asana-patterns.md`** for the section move API pattern.

**Post a start comment** on the task with the branch name and draft PR URL (deduplicate by checking for existing 🏁 comment for this branch). If the move fails, report why but do not block the workflow.

### Step 10: Route to the Right Workflow

Compile full task context (name, notes, custom fields, task ID, subtasks, comments, attachments, branch name) and route based on **Category** custom field:

- **"Bug"** — Follow the verify → fix → verify loop (Steps 10a–10c below).
- **Anything else** (Feature Request, Tech Debt, etc.):
  - If `$ARGUMENTS` contains `brainstorm` — invoke `superpowers:brainstorming` with the full context.
  - If `$ARGUMENTS` contains `feature-dev` — invoke `feature-dev:feature-dev` with the full context.
  - If no workflow argument was provided — ask (blocking):
    > "How do you want to approach this?
    > 1. Brainstorm the design first (`superpowers:brainstorming`)
    > 2. Go straight to implementation (`feature-dev:feature-dev`)"
    Wait for explicit answer before routing. No default assumed.
- **Category missing** — Prompt: "Is this a bug fix or a feature?" then apply the routing above.

The branch is already created and checked out — the downstream skill works on it directly.

### Step 10a: Verify Bug (Bug category only)

Invoke `web-qa` in **investigate** mode with the bug description from the Asana ticket as the question and the SUT URL (if known from CLAUDE.md or task notes).

- **Confirmed** (bug reproduced with evidence) → web-qa posts the QA report to the Asana task (Step 6 in the generic-qa process). Proceed to Step 10b, passing the full report as context.
- **Cannot reproduce** → **stop**. Tell the operator the bug could not be reproduced. Let them decide: fix SUT setup, clarify the bug description, or skip verification and proceed to debugging anyway.

### Step 10b: Fix Bug

Invoke `fix-bug` with the web-qa report as enriched context. This gives the debugger richer context than the ticket alone — reproduction steps, evidence, and root cause analysis from runtime observation.

### Step 10c: Verify Fix

After the fix is committed, re-invoke `web-qa` in **verify** mode with the original reproduction steps from Step 10a.

- **Pass** (behavior now matches expected) → confirmed fixed, proceed to Step 11.
- **Fail** (behavior still matches original actual) → tell the operator the fix didn't resolve the issue. Return to Step 10b for another debugging pass.

**Handoff instruction:** When passing context to the downstream skill, include this explicitly:
> "When this workflow is complete, return to `start-task` Step 11 and invoke `ship-it`. Do not end the session — there is one more step."

This ensures the downstream skill knows control must return here rather than closing out.

### Step 11: Ship It

**This step runs as soon as the development workflow signals completion** — `feature-dev` at Phase 7 (Summary), or `systematic-debugging` after confirming the fix is verified. Do not wait for the user to ask.

Invoke `ship-it`. The following context is already in this session — pass it through, do not re-ask:

| What | Source |
|------|--------|
| Task GID | Extracted in Step 1 |
| Task URL | Provided in `$ARGUMENTS` |
| Task ID | From task custom fields (Step 2) |
| Branch name | Created in Step 7 |
| Draft PR URL | Captured in Step 8 |
| Sprint project GID | From task memberships (Step 2) |
| Section mappings | Discovered when moving to "In Progress" (Step 9) |

`ship-it` will run pre-ship-check, generate a work summary, promote the draft PR to ready, move the Asana task to "In Review", and post a completion comment.

## Pause Flow

Triggered when the user says "park this", "I'm blocked", "pause task", or similar during any phase of work. Commits WIP, drafts a blocking question for user approval, posts to Asana, saves a checkpoint, and pushes. See **`references/checkpoints.md`** for the full pause flow, checkpoint file format, and trigger phrases.

## Important Notes

- This skill orchestrates the full lifecycle: start → develop → ship. It hands off to `ship-it` when development is done.
- Include the task ID in branch names and commit messages for traceability.
- Route all Asana API calls through the `asana-api` skill — no raw curl.
- If `$ASANA_PERSONAL_ACCESS_TOKEN` is not set, stop and guide configuration before proceeding.

## Reference Files

- **`references/skill-dependencies.md`** — External plugin dependencies, install commands, check instructions
- **`references/validation-rules.md`** — Sprint-readiness checks, failure display, skip rules
- **`references/asana-patterns.md`** — URL formats, API fields, section moves, comment posting
- **`references/git-workflow.md`** — Existing work detection, branch creation, naming convention
- **`references/checkpoints.md`** — Checkpoint file format, pause flow, resume flow, edge cases
