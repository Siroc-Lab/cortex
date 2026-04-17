---
name: start-task-steps
description: >
  This skill should be used when the user provides an Asana task URL and wants to begin working on it
  with full step-by-step checkpoint tracking — "start task", "work on this", "pick up this ticket",
  "let's start on [task-id]", "jump on this", "begin this task", or pastes an Asana URL with intent
  to start development. Orchestrates the full lifecycle: validates sprint-readiness, sets up the branch
  and draft PR, routes to the right development workflow (feature-dev, fix-bug, or brainstorm), and
  ships via ship-it when done. Every step writes a checkpoint file so work can be paused and resumed
  at any point. Also handles pausing ("park this", "I'm blocked", "pause task", "put this on hold")
  and resuming ("resume task", "pick up where I left off", "continue [task-id]"). Prefer this variant
  over start-task for long, complex, or interruptible tasks.
argument-hint: <asana-task-url> [brainstorm|feature-dev]
---

# Start Task (Steps)

Take an Asana task, validate it's ready for development, understand the work, set up the branch, and route to the right skill. This skill is the conductor — it validates, prepares, and hands off.

Every step updates the checkpoint file immediately upon completion. This is mandatory. The checkpoint is the single source of truth for task progress.

## Checkpoint Protocol

The checkpoint file at `.claude/checkpoints/<task-gid>.md` tracks every step of this skill's execution. It must be written after each step. This is not optional and cannot be skipped.

**The rule:** A step is not complete until its row in the checkpoint file is updated. You may not begin the next step until the file is written. The sequence for every step is:

```
1. Mark the step row: State → in_progress, Attempts +1, update last_updated
2. Do the step's work
3. Mark the step row: Completed → [x], State → completed, fill Comment and Auto
4. Only then: move to the next step
```

If a step fails or blocks, set State → `blocked` and follow the Pause Flow. Never leave a step as `in_progress` and advance past it.

## Prerequisites

- `asana-api` skill for all Asana API operations — handles token resolution and setup guidance.
- Access to `feature-dev:feature-dev`, `superpowers:systematic-debugging`, and optionally `superpowers:brainstorming` skills (required if using the `brainstorm` workflow for non-bug tasks)
- `fix-bug` is bundled — no external dependency for bug tasks
- The `asana-api` skill for all Asana API operations

## Reference Files

- **`references/skill-dependencies.md`** — External plugin dependencies, install commands, check instructions
- **`references/checkpoints.md`** — Checkpoint file format, step update rules, pause/resume flows
- **`references/validation-rules.md`** — Sprint-readiness checks, failure display, skip rules
- **`references/asana-patterns.md`** — URL formats, API fields, section moves, comment posting
- **`references/git-workflow.md`** — Existing work detection, branch creation, naming convention

---

## The Flow

### Step 0: Initialize Checkpoint

**This is the very first action. Do not call any API or run any git command before this.**

0. Check external skill dependencies before creating the checkpoint. These are **not bundled** with this plugin:
   - **`feature-dev@claude-plugins-official`** — required for non-bug tasks using the `feature-dev` workflow
   - **`superpowers@claude-plugins-official`** — required for Bug tasks (`fix-bug` uses `systematic-debugging`) and for non-bug tasks using the `brainstorm` workflow

   If either is missing, warn the user and ask whether to install now or continue. This is an **advisory blocking step** — wait for the user's answer before proceeding. See **`references/skill-dependencies.md`** for check instructions, install commands, and warning message templates.

1. Extract the task GID from the URL in `$ARGUMENTS`. See `references/asana-patterns.md` for URL formats. If no URL is provided, prompt for it now.

2. Check for an existing checkpoint:
   ```bash
   ls .claude/checkpoints/<task-gid>.md 2>/dev/null
   ```

3. **If checkpoint found** → this is a resume. Read `references/checkpoints.md` Section 5 for the resume flow. Find the first row where `Completed = [ ]` and jump to that step. Do not re-run completed steps.

4. **If no checkpoint** → create it now:
   ```bash
   mkdir -p .claude/checkpoints/
   ```
   Write the template from `references/checkpoints.md` Section 1 with `task_gid`, `asana_url`, `created_at`, `last_updated` filled in. All steps: `[ ]`, state `—`, attempts `0`. Ensure `.claude/checkpoints/` is in `.gitignore`.

---

### Step 1: Get the Asana Task URL

1. Open checkpoint: set Step 1 → `in_progress`, Attempts → 1, update `last_updated`.
2. Confirm the URL is valid and the GID is numeric (already extracted in Step 0).
3. Write checkpoint: Step 1 → `[x]` | `completed` | Auto `[x]` | Comment: `GID: <task-gid>`.

---

### Step 2: Fetch Task Details

1. Open checkpoint: set Step 2 → `in_progress`, Attempts +1, update `last_updated`.
2. Fetch the full task via the `asana-api` skill with all required `opt_fields` (see `references/asana-patterns.md`).
3. Present a quick summary: task name, assignee, category, task ID, sprint, and backlog board memberships. Classify memberships per `plugins/asana-workflow/references/board-resolution.md`.
4. Write checkpoint: Step 2 → `[x]` | `completed` | Auto `[x]` | Comment: `<task-id> — <task-name>`. Update frontmatter: `task_id`.

---

### Step 3: Validate Sprint-Readiness

1. Open checkpoint: set Step 3 → `in_progress`, Attempts +1, update `last_updated`.
2. Run four validation checks: Active sprint membership, Estimated time, Product Status = Assigned, ID field. See `references/validation-rules.md` for details. The validation-rules reference loads the board registry cache (see `plugins/asana-workflow/references/board-resolution.md`) to resolve the active sprint.
3. Report failures as a checklist. Active sprint membership, Estimated time, and Product Status are blocking — offer to set the latter two via API. Only the ID field can be skipped. Do not proceed until all blocking checks pass.
4. If checks require fixing (user interaction), re-run this step after each fix (return to sub-step 1, incrementing Attempts again).
5. Write checkpoint: Step 3 → `[x]` | `completed` | Auto `[ ]` | Comment: `All checks passed` or `Fixed: <what was set>`.

---

### Step 4: Fetch Subtasks

1. Open checkpoint: set Step 4 → `in_progress`, Attempts +1, update `last_updated`.
2. Fetch subtasks via the `asana-api` skill. Group by status: incomplete = remaining work, completed = already done. Include in downstream context.
3. Write checkpoint: Step 4 → `[x]` | `completed` | Auto `[x]` | Comment: `<N> subtasks (<M> complete, <K> remaining)`.

---

### Step 5: Fetch Comments and Attachments

1. Open checkpoint: set Step 5 → `in_progress`, Attempts +1, update `last_updated`.
2. Fetch task stories and filter for `type: "comment"`. List attachments by name, noting any images (mockups, screenshots). See `references/asana-patterns.md` for details.
3. Write checkpoint: Step 5 → `[x]` | `completed` | Auto `[x]` | Comment: `<N> comments, <M> attachments`.

---

### Step 6: Check for Existing Work

1. Open checkpoint: set Step 6 → `in_progress`, Attempts +1, update `last_updated`.
2. Check if work already exists for this task ID (see `references/git-workflow.md` for detection commands).
3. If a branch or PR exists, offer to resume that work or start fresh. If resuming, check out the existing branch and skip Step 7.
4. Write checkpoint: Step 6 → `[x]` | `completed` | Auto `[x]` | Comment: `No existing branch` or `Resumed: <branch-name>`.

---

### Step 6b: Ask About Worktree (BLOCKING)

Skip this step if Step 6 found an existing branch to resume.

1. Open checkpoint: set Step 6b → `in_progress`, Attempts +1, update `last_updated`.
2. Ask the user whether to use a git worktree. This is a **blocking** question — wait for an explicit answer before proceeding.

   > Would you like to work in a git worktree (isolated copy of the repo) or directly in the current directory?
   > - **Worktree** _(recommended for parallel work — keeps main directory clean)_
   > - **Current directory**

   If the user chooses worktree, use `EnterWorktree` to create an isolated copy. The branch will be created inside the worktree in Step 7.

3. Write checkpoint: Step 6b → `[x]` | `completed` | Auto `[ ]` | Comment: `worktree` or `current directory`.

---

### Step 6c: Confirm Base Branch (BLOCKING)

Skip this step if Step 6 found an existing branch to resume.

1. Open checkpoint: set Step 6c → `in_progress`, Attempts +1, update `last_updated`.
2. Ask the user which branch to base the new branch on. This is a **blocking** question — wait for an explicit answer before proceeding.

   > Which branch should `<task-id>/<slug>` be based on?
   > - **main** _(default — latest stable base)_
   > - Another branch _(enter branch name)_

   Default to `main` only after the user confirms. Record the chosen base branch for Step 7.

3. Write checkpoint: Step 6c → `[x]` | `completed` | Auto `[ ]` | Comment: `<base-branch>`.

---

### Step 7: Create Feature Branch

Skip this step if Step 6 found an existing branch to resume — go straight to writing the checkpoint for Step 7 as `completed | Skipped: resumed existing branch`.

1. Open checkpoint: set Step 7 → `in_progress`, Attempts +1, update `last_updated`.
2. Create a branch using the task ID and a slug from the task name. Use the **base branch confirmed in Step 6c** (not assumed `main`). Inform (do not ask) when creating. See `references/git-workflow.md` for commands and naming convention.
3. Write checkpoint: Step 7 → `[x]` | `completed` | Auto `[x]` | Comment: `<branch-name> off <base>`. Update frontmatter: `branch`, `base_branch`.

---

### Step 8: Create Draft PR

1. Open checkpoint: set Step 8 → `in_progress`, Attempts +1, update `last_updated`.
2. Create an empty commit:
   ```bash
   git commit --allow-empty -m "<task-id> :: <task-name-slug> (start)"
   ```
3. Push the branch:
   ```bash
   git push -u origin $(git branch --show-current)
   ```
4. Create the draft PR:
   ```bash
   gh pr create --draft --title "<TASK-ID> :: <description>" --body "$(cat <<'EOF'
   ## Asana Task
   <asana-task-url>
   EOF
   )"
   ```
5. Capture the draft PR URL — needed for the Asana start comment and `ship-it`.
6. Write checkpoint: Step 8 → `[x]` | `completed` | Auto `[x]` | Comment: `<pr-url>`.

---

### Step 9: Move to In Progress

1. Open checkpoint: set Step 9 → `in_progress`, Attempts +1, update `last_updated`.
2. Move the task to "In Progress" on the Sprint board. Skip if already there. See `references/asana-patterns.md` for the section move API pattern. If the move fails, report why but continue.
3. Write checkpoint: Step 9 → `[x]` | `completed` | Auto `[x]` | Comment: `Moved` or `Already in progress` or `Failed: <reason>`.

---

### Step 10: Post Start Comment

1. Open checkpoint: set Step 10 → `in_progress`, Attempts +1, update `last_updated`.
2. Post a start comment on the Asana task with the branch name and draft PR URL. Deduplicate by checking for an existing 🏁 comment for this branch. See `references/asana-patterns.md` for the comment format.
3. Write checkpoint: Step 10 → `[x]` | `completed` | Auto `[x]` | Comment: `Posted` or `Skipped (duplicate)`.

---

### Step 11: Route to the Right Workflow

1. Open checkpoint: set Step 11 → `in_progress`, Attempts +1, update `last_updated`.
2. Compile full task context (name, notes, custom fields, task ID, subtasks, comments, attachments, branch name) and route based on **Category** custom field:
   - **"Bug"** → Invoke `fix-bug` with the full task context. `fix-bug` is bundled — no dependency check needed.
   - **Anything else** (Feature Request, Tech Debt, etc.):
     - If `$ARGUMENTS` contains `brainstorm` — invoke `superpowers:brainstorming` with the full context.
     - If `$ARGUMENTS` contains `feature-dev` — invoke `feature-dev:feature-dev` with the full context.
     - If no workflow argument was provided — ask (blocking):
       > "How do you want to approach this?
       > 1. Brainstorm the design first (`superpowers:brainstorming`)
       > 2. Go straight to implementation (`feature-dev:feature-dev`)"
       Wait for explicit answer before routing. No default assumed.
   - **Category missing** → Prompt: "Is this a bug fix or a feature?" then apply the routing above.
3. Include this in the handoff context passed to the downstream skill:
   > "When this workflow is complete, return to `start-task-steps` Step 12 and invoke `ship-it`. Do not end the session — there is one more step."
4. Write checkpoint: Step 11 → `[x]` | `completed` | Auto `[ ]` if user input was needed, `[x]` otherwise | Comment: `fix-bug`, `brainstorm`, or `feature-dev`. Update frontmatter: `workflow`.

---

### Step 12: Ship It

**This step runs as soon as the development workflow signals completion** — `fix-bug` after the fix is verified, `feature-dev` at Phase 7 (Summary), or `superpowers:brainstorming` when the design is done. Do not wait for the user to ask.

1. Open checkpoint: set Step 12 → `in_progress`, Attempts +1, update `last_updated`.
2. Invoke `ship-it`. The following context is already in this session — pass it through, do not re-ask:

   | What | Source |
   |------|--------|
   | Task GID | Step 1 (checkpoint frontmatter) |
   | Task URL | `$ARGUMENTS` / checkpoint frontmatter |
   | Task ID | Step 2 (checkpoint frontmatter: `task_id`) |
   | Branch name | Step 7 (checkpoint frontmatter: `branch`) |
   | Draft PR URL | Step 8 (checkpoint comment) |
   | Sprint project GID | Board cache `active_sprint.gid` (loaded in Step 3) |
   | Section mappings | Step 9 (discovered when moving to "In Progress") |

   `ship-it` will run pre-ship-check, generate a work summary, promote the draft PR to ready, move the Asana task to "In Review", and post a completion comment.

3. Write checkpoint: Step 12 → `[x]` | `completed` | Auto `[x]` | Comment: `Shipped: <pr-url>`.

---

## Pause Flow

Triggered when the user says "park this", "I'm blocked", "pause task", or similar during any step.

Set the current step's State → `blocked` in the checkpoint before doing anything else. Then follow `references/checkpoints.md` Section 4: commit WIP, draft blocking question for user approval, post to Asana, push.

## Important Notes

- This skill orchestrates the full lifecycle: start → develop → ship. It hands off to `ship-it` when development is done (Step 12).
- Include the task ID in branch names and commit messages for traceability.
- Route all Asana API calls through the `asana-api` skill — no raw curl.
