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
  and resuming ("resume task", "pick up where I left off", "continue [task-id]"). For long, complex,
  or interruptible tasks that need step-by-step checkpoint tracking, add "steps" to the arguments —
  "start task steps", "with checkpoints". For a shortcut that runs the full Asana orchestration and
  ship-it but skips sub-skill routing (feature-dev, brainstorming, fix-bug) and implements inline
  instead, add "fast" to the arguments — "start task fast", "fast mode", "just start coding".
argument-hint: <asana-task-url> [brainstorm|feature-dev|fast] [steps]
---

# Start Task

Take an Asana task, validate it's ready for development, understand the work, set up the branch, and route to the right skill. This skill is the conductor — it validates, prepares, and hands off.

## Prerequisites

- `asana-api` skill for all Asana API operations — handles token resolution and setup guidance.
- Access to `feature-dev:feature-dev`, `superpowers:systematic-debugging`, and optionally `superpowers:brainstorming` skills (external — see `references/skill-dependencies.md`).

## Argument Parsing

Parse `$ARGUMENTS` once and establish these flags. The rest of the skill refers to them by name instead of re-parsing.

| Flag | Set when `$ARGUMENTS` contains | Effect |
|------|-------------------------------|--------|
| `fast_mode` | `fast` | Step 10 skips sub-skill routing and the Step 11 QA sub-flow; implements inline |
| `steps_mode` | `steps` | Mandatory per-step checkpoint bookkeeping (see Steps Mode below) |
| `workflow_choice` | `brainstorm` or `feature-dev` | Non-bug routing at Step 10; if neither, Step 10 asks the operator |

`fast_mode` is mutually exclusive with `workflow_choice` (fast skips routing entirely). `steps_mode` is orthogonal to both.

If `steps_mode` is set, initialize the checkpoint per **`references/checkpoints-steps.md`** → "Initialization" (or resume from it) now, before Step 0.

## Fast Mode

Set when `fast_mode` flag is active (see Argument Parsing above).

Fast mode runs the full lifecycle (Steps 0–9 and Step 12) unchanged but replaces Step 10 skill routing with direct inline implementation, and skips Step 11 (QA sub-flow) entirely. No `feature-dev`, `brainstorming`, `fix-bug`, or QA skill is invoked — implement the solution immediately using built-in tools (Read, Edit, Bash, Grep, etc.) and reason about it directly in this conversation.

**What is skipped:** only the sub-skill routing in Step 10 and the entire Step 11 QA sub-flow. Everything else — dependency checks, sprint validation, branch creation, draft PR, Asana status move/comment, and the ship-it handoff — runs as normal.

## Steps Mode

Set when `steps_mode` flag is active (see Argument Parsing above).

Steps mode does not change the flow — every step below runs as normal. It only adds mandatory checkpoint bookkeeping so work can be paused and resumed at any point without losing progress.

**The rule:** when steps mode is active, a step is not complete until its row in the checkpoint file is updated. For every step:

1. Mark the row: `State` → `in_progress`, `Attempts` +1, update `last_updated`.
2. Do the step's work.
3. Mark the row: `Completed` → `[x]`, `State` → `completed`, fill `Comment` and `Auto`.
4. Only then: move to the next step.

If a step fails or blocks, set `State` → `blocked` and follow the Pause Flow. If a step's preconditions don't apply in this run (wrong category, `qa-skill=none`, fast mode, operator opted out), mark it as skipped: `Completed` → `[~]`, `State` → `skipped`, `Comment` → the reason. A `[~]` row is terminal — treated like `[x]` on resume. Never leave a step as `in_progress` and advance past it.

Before anything else, initialize the checkpoint file at `.claude/checkpoints/<task-gid>.md`. If a checkpoint already exists, load it and resume from the first incomplete row instead of re-running completed steps. See **`references/checkpoints-steps.md`** for the file format, steps table, comment conventions, and initialization/update rules; **`references/checkpoints.md`** for the shared pause and resume flows.

Steps mode is orthogonal to `fast_mode` and `workflow_choice` — it can be combined with any of them.

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

Present a quick summary for confirmation: task name, assignee, category, task ID, sprint, and backlog board memberships. Classify memberships per **`plugins/asana-workflow/references/board-resolution.md`**.

### Step 3: Validate Sprint-Readiness

Run four validation checks: Active sprint membership, Estimated time, Product Status = Assigned, and ID field presence. See **`references/validation-rules.md`** for check details, failure display format, fix-offer logic, and skip rules. The validation-rules reference loads the board registry cache (see **`plugins/asana-workflow/references/board-resolution.md`**) to resolve the active sprint.

Report failures as a checklist. Active sprint membership, Estimated time, and Product Status are all blocking — offer to set the latter two via API, but do not proceed until all three pass. Only the ID field can be skipped after a warning.

### Step 4: Fetch Subtasks

Fetch subtasks via the `asana-api` skill. Group by status (incomplete = remaining work, completed = already done). Include subtasks in downstream context so the receiving skill understands what "done" looks like.

### Step 5: Fetch Comments and Attachments

Fetch task stories and filter for comments. List attachments by name, noting any images (mockups, screenshots). See **`references/asana-patterns.md`** for details.

### Step 6: Check for Existing Work

Before creating a branch, check if work already exists for this task ID. See **`references/git-workflow.md`** for the detection commands.

If a branch or PR exists, offer to resume or start fresh. If resuming, check out the existing branch and skip creation.

### Step 6a: Check for Checkpoint (Resume)

**Skip this step in steps mode** — the Initialization section of `references/checkpoints-steps.md` already handled checkpoint detection before Step 0.

After fetching remote refs, check for `.claude/checkpoints/<task-gid>.md`. If found, this is a resume — present the checkpoint state, check for new Asana comments since the pause, and offer to resume. The checkpoint format is detected from its contents (presence of a `## Steps` table = steps mode, narrative body = default mode); resume honors the file's original mode regardless of the current `$ARGUMENTS`. See **`references/checkpoints.md`** for the full resume flow and edge cases (deleted branch, completed task, no answer yet).

On resume, skip validation and branch creation — check out the existing branch and route directly to the workflow specified in the checkpoint. The same Step 12 ship-it handoff applies after the workflow completes.

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

### Step 9a: Move to In Progress

**This happens automatically — no permission needed.**

Move the task to "In Progress" on the Sprint board. Skip if already there. See **`references/asana-patterns.md`** for the section move API pattern. If the move fails, report why but do not block the workflow — proceed to Step 9b.

### Step 9b: Post Start Comment

**This happens automatically — no permission needed.**

Post a start comment on the task with the branch name and draft PR URL. Deduplicate by checking for an existing 🏁 comment for this branch. See **`references/asana-patterns.md`** for the comment format.

### Step 10: Route to the Right Workflow

Compile full task context (name, notes, custom fields, task ID, subtasks, comments, attachments, branch name) and route based on **Category** custom field:

**If `fast_mode`** — skip all skill routing regardless of category. Implement the solution directly in this conversation using built-in tools (Read, Edit, Bash, Grep, etc.). Do not invoke `feature-dev`, `brainstorming`, `fix-bug`, or any QA skill. Skip Step 11 (QA sub-flow) entirely and proceed to Step 12 when done.

**Otherwise:**

- **"Bug"** — Follow the QA sub-flow: verify → fix → verify loop.
- **Anything else** (Feature Request, Tech Debt, etc.):
  - If `workflow_choice` is `brainstorm` — invoke `superpowers:brainstorming` with the full context.
  - If `workflow_choice` is `feature-dev` — invoke `feature-dev:feature-dev` with the full context.
  - If `workflow_choice` is unset — ask (blocking):
    > "How do you want to approach this?
    > 1. Brainstorm the design first (`superpowers:brainstorming`)
    > 2. Go straight to implementation (`feature-dev:feature-dev`)"
    Wait for explicit answer before routing. No default assumed.
  - **Handoff instruction:** When passing context to `feature-dev` or `brainstorming`, include:
    > "When this workflow is complete, return to `start-task` for non-bug QA verification and the ship-it handoff. Do not end the session — there are more steps."
- **Category missing** — Prompt: "Is this a bug fix or a feature?" then apply the routing above.

The branch is already created and checked out — the downstream skill works on it directly.

### Step 11: QA Sub-flow

Run the QA sub-flow per **`plugins/asana-workflow/references/qa-routing.md`** (bug: verify → fix → verify loop; non-bug: hard-gated operator prompt; `qa-skill=none` handled internally). For non-bug tasks this runs after the development workflow returns; for bug tasks it runs immediately after routing.

**In fast mode** the sub-flow is skipped entirely. In steps-mode checkpoints, mark every QA row `[~]` / `skipped` / `fast mode`.

### Step 12: Ship It

**This step runs after the bug QA verify loop (for bugs) or the non-bug QA verification (for non-bugs) completes — or when the operator skips QA.** Do not wait for the user to ask.

Invoke `ship-it`. The following context is already in this session — pass it through, do not re-ask:

| What | Source |
|------|--------|
| Task GID | Extracted in Step 1 |
| Task URL | Provided in `$ARGUMENTS` |
| Task ID | From task custom fields (Step 2) |
| Branch name | Created in Step 7 |
| Draft PR URL | Captured in Step 8 |
| Sprint project GID | From board cache `active_sprint.gid` (loaded in Step 3) |
| Section mappings | Discovered when moving to "In Progress" (Step 9a) |

`ship-it` will run pre-ship-check, generate a work summary, promote the draft PR to ready, move the Asana task to "In Review", and post a completion comment.

**Steps mode only:** after `ship-it` returns successfully, delete `.claude/checkpoints/<task-gid>.md`. Post-ship work (code-review fixes, follow-up commits) is out of scope for this checkpoint — see **`references/checkpoints-steps.md`** → "Lifecycle End".

## Pause Flow

Triggered when the user says "park this", "I'm blocked", "pause task", or similar during any phase of work. Commits WIP, drafts a blocking question for user approval, posts to Asana, saves/updates a checkpoint, and pushes. See **`references/checkpoints.md`** for the full pause flow and trigger phrases; the mode-specific file format lives in `references/checkpoints-steps.md` or `references/checkpoints-pause.md`.

## Important Notes

- This skill orchestrates the full lifecycle: start → develop → ship. It hands off to `ship-it` when development is done.
- Include the task ID in branch names and commit messages for traceability.
- Route all Asana API calls through the `asana-api` skill — no raw curl.

## Reference Files

- **`references/skill-dependencies.md`** — External plugin dependencies, install commands, check instructions
- **`references/validation-rules.md`** — Sprint-readiness checks, failure display, skip rules
- **`references/asana-patterns.md`** — URL formats, API fields, section moves, comment posting
- **`references/git-workflow.md`** — Existing work detection, branch creation, naming convention
- **`references/checkpoints.md`** — Shared entry: file location, frontmatter, mode detection, pause flow, resume flow, edge cases
- **`references/checkpoints-steps.md`** — Steps-mode specifics: steps table, initialization, per-step updates, lifecycle end
- **`references/checkpoints-pause.md`** — Default-mode specifics: narrative template, pause-only file creation
- **`plugins/asana-workflow/references/qa-routing.md`** — QA skill resolution and the QA sub-flow (plugin-level shared reference with pre-ship-check)
