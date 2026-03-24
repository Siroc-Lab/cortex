---
name: start-task
description: >
  This skill should be used when the user provides an Asana task URL and wants to begin working on it,
  or says things like "start task", "work on this", "pick up this ticket", "begin this task", or pastes
  an Asana URL with intent to start development. Also handles pausing blocked work ("park this",
  "I'm blocked", "pause task", "put this on hold") and resuming paused tasks ("resume task",
  "pick up where I left off", "continue [task-id]"). This is the entry point for the company's
  development workflow — from ticket to code, with checkpoint support for blocked work.
argument-hint: <asana-task-url>
---

# Start Task

Take an Asana task, validate it's ready for development, understand the work, set up the branch, and route to the right skill. This skill is the conductor — it validates, prepares, and hands off.

## Prerequisites

- `$PERSONAL_ACCESS_TOKEN` env var set in `~/.zshrc` — the Asana personal access token. If missing, stop and guide setup:
  > Add to `~/.zshrc`: `export PERSONAL_ACCESS_TOKEN="your-asana-token-here"`
  > Get token from: https://app.asana.com/0/my-apps
- Access to `feature-dev:feature-dev` and `superpowers:systematic-debugging` skills
- The `asana-api` skill for all Asana API operations

## The Flow

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

On resume, skip validation and branch creation — check out the existing branch and route directly to the workflow specified in the checkpoint.

### Step 7: Create Feature Branch

Create a branch using the task ID and a slug from the task name. Default to `main` as base. Inform (do not ask) when creating. See **`references/git-workflow.md`** for commands and naming convention.

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

- **"Bug"** — Invoke `superpowers:systematic-debugging` with the full context as the bug report.
- **Anything else** (Feature Request, Tech Debt, etc.) — Invoke `feature-dev:feature-dev` with the full context as the feature specification.
- **Category missing** — Prompt: "Is this a bug fix or a feature?"

The branch is already created and checked out — the downstream skill works on it directly.

## Pause Flow

Triggered when the user says "park this", "I'm blocked", "pause task", or similar during any phase of work. Commits WIP, drafts a blocking question for user approval, posts to Asana, saves a checkpoint, and pushes. See **`references/checkpoints.md`** for the full pause flow, checkpoint file format, and trigger phrases.

## Important Notes

- This skill starts work. It does not ship it. Shipping is handled by `ship-it`.
- Include the task ID in branch names and commit messages for traceability.
- Route all Asana API calls through the `asana-api` skill — no raw curl.
- If `$PERSONAL_ACCESS_TOKEN` is not set, stop and guide configuration before proceeding.

## Reference Files

- **`references/validation-rules.md`** — Sprint-readiness checks, failure display, skip rules
- **`references/asana-patterns.md`** — URL formats, API fields, section moves, comment posting
- **`references/git-workflow.md`** — Existing work detection, branch creation, naming convention
- **`references/checkpoints.md`** — Checkpoint file format, pause flow, resume flow, edge cases
