# Checkpoints Reference

Checkpoints let `start-task` pause and resume work without losing context. There are two levels of tracking:

- **Default (no `steps`):** a checkpoint is only written when the user pauses via the Pause Flow. Section 1's full step table is not required — a lightweight frontmatter + notes body is enough.
- **Steps mode (`$ARGUMENTS` contains `steps`):** the checkpoint is initialized on entry and updated around every step. The step table in Section 1 is the single source of truth for task progress.

Both modes share the same file location, pause flow, and resume flow.

---

## Section 1: File Format

**Location:** `.claude/checkpoints/<task-gid>.md`

Use the Asana task GID (numeric) as the filename — it is available from the URL before any API call is made. Create the directory on first use: `mkdir -p .claude/checkpoints/`. Ensure `.claude/checkpoints/` is in `.gitignore` — these are local-only state files and must not be committed.

**Never overwrite an existing checkpoint on initialization.** If the file exists, work was already started — load it and resume.

### Frontmatter Fields

| Field | Description |
|-------|-------------|
| `task_gid` | Asana task GID (numeric, from URL) |
| `task_id` | Project ID custom field (e.g., `MT251-47`) — filled in after Step 2 |
| `asana_url` | Full Asana task URL |
| `branch` | Feature branch name — filled in after Step 7 |
| `base_branch` | Branch it was created from — filled in after Step 7 |
| `workflow` | `fix-bug`, `brainstorm`, `feature-dev`, or `fast` — filled in after Step 10 |
| `created_at` | ISO 8601 timestamp — set once on initialization |
| `last_updated` | ISO 8601 timestamp — updated after every step (steps mode) or on pause (default mode) |
| `paused_at` | ISO 8601 timestamp — set when entering the Pause Flow; used to filter new Asana comments on resume |
| `blocked_on` | Name of the person who should answer (informational) — set when blocked |

### Steps Table (steps mode)

Six-column markdown table tracking every step of the workflow.

| Column | Values | Description |
|--------|--------|-------------|
| **Step** | Text | Step number and name |
| **Completed** | `[ ]` / `[x]` / `[~]` | `[x]` done, `[~]` skipped as not-applicable, `[ ]` still open |
| **Comment** | Text | Key data captured or what happened |
| **Attempts** | Integer | How many times this step has been attempted (starts at 0) |
| **State** | `—` / `in_progress` / `completed` / `blocked` / `skipped` | Current execution state |
| **Auto** | `[ ]` / `[x]` | `[x]` if completed without user input; `[ ]` if user approval was required |

### Template (New Checkpoint, steps mode)

```markdown
---
task_gid: ""
task_id: ""
asana_url: ""
branch: ""
base_branch: ""
workflow: ""
created_at: "<iso8601>"
last_updated: "<iso8601>"
---

## Steps

| Step | Completed | Comment | Attempts | State | Auto |
|------|-----------|---------|----------|-------|------|
| 0. Dependency Check | [ ] | | 0 | — | [ ] |
| 1. Get Task URL | [ ] | | 0 | — | [x] |
| 2. Fetch Task Details | [ ] | | 0 | — | [x] |
| 3. Validate Sprint-Readiness | [ ] | | 0 | — | [ ] |
| 4. Fetch Subtasks | [ ] | | 0 | — | [x] |
| 5. Fetch Comments & Attachments | [ ] | | 0 | — | [x] |
| 6. Check Existing Work | [ ] | | 0 | — | [x] |
| 6b. Ask About Worktree | [ ] | | 0 | — | [ ] |
| 6c. Confirm Base Branch | [ ] | | 0 | — | [ ] |
| 7. Create Feature Branch | [ ] | | 0 | — | [x] |
| 8. Create Draft PR | [ ] | | 0 | — | [x] |
| 9a. Move to In Progress | [ ] | | 0 | — | [x] |
| 9b. Post Start Comment | [ ] | | 0 | — | [x] |
| 10. Route to Workflow | [ ] | | 0 | — | [ ] |
| QA: Resolve | [ ] | | 0 | — | [ ] |
| QA: Investigate Bug | [ ] | | 0 | — | [ ] |
| QA: Fix Bug | [ ] | | 0 | — | [x] |
| QA: Verify Fix | [ ] | | 0 | — | [ ] |
| QA: Verify Non-Bug | [ ] | | 0 | — | [ ] |
| 12. Ship It | [ ] | | 0 | — | [x] |

## Notes
```

The `QA:` rows are only exercised when their preconditions apply (bug vs non-bug, resolved QA skill != `none`, not in fast mode). Leave unreached rows as `[ ]`, state `—` if the skill doesn't get a chance to set them — but prefer marking them `[~]` / `skipped` with a reason when the non-applicability is known at the time the row would run (see Section 3).

### Template (Default Mode — Pause Only)

When steps mode is not active, the checkpoint is created at pause time with only frontmatter and a narrative body. No Steps table is required.

```markdown
---
task_id: MT251-47
task_gid: "1209384756102938"
asana_url: https://app.asana.com/0/123456789/1209384756102938
branch: MT251-47/add-csv-export
base_branch: main
workflow: feature-dev
paused_at: 2026-03-24T14:32:00Z
blocked_on: Sarah Chen
---

## Blocked On

Waiting for Sarah Chen to confirm whether the CSV export should use the
user's local timezone or UTC. This affects how timestamps are formatted
in the export output.

## Progress

- Completed discovery and schema analysis
- Implemented CSV serialization for all non-timestamp fields
- Timestamp formatting logic is stubbed — pending answer above

## Context for Resume

The relevant code is in `src/exporters/csv.ts`. The stub is marked with
`// TODO: awaiting timezone decision`. Once answered, update
`formatTimestamp()` and add corresponding tests in `csv.test.ts`.
```

---

## Section 2: Initialization (Steps Mode Only)

**This is the very first action `start-task` takes when `steps` is in `$ARGUMENTS`** — before any Asana API call, before any git command.

1. **Extract task GID** from the URL in `$ARGUMENTS`. If no URL is provided, prompt for it now.

2. **Check for an existing checkpoint:**
   ```bash
   ls .claude/checkpoints/<task-gid>.md 2>/dev/null
   ```

3. **If checkpoint found** → this is a resume. Detect the file's mode (Section 5 "Mode Detection") and go to **Section 5: Resume Flow**. If the file is default-mode, resume proceeds as default mode even though the operator passed `steps` — the file's mode is authoritative.

4. **If no checkpoint** → create it now:
   ```bash
   mkdir -p .claude/checkpoints/
   ```
   Write the steps-mode template with `task_gid`, `asana_url`, `created_at`, `last_updated` filled in. All steps: `[ ]`, state `—`, attempts `0`. Add `.claude/checkpoints/` to `.gitignore` if not already present.

In default mode, no checkpoint is created on entry. A checkpoint is only written if the user pauses (Section 4).

---

## Section 3: Step Updates (Steps Mode Only)

**After every step, update the checkpoint immediately. This is not optional.**

### Update Pattern

When a step **starts**:
- Set `State` → `in_progress`
- Increment `Attempts` by 1
- Update `last_updated` to now

When a step **completes successfully**:
- Set `Completed` → `[x]`
- Set `State` → `completed`
- Set `Comment` → key data (see table below)
- Set `Auto` → `[x]` if no user input was needed; `[ ]` if user approved/decided something
- Update `last_updated` to now

When a step is **blocked**:
- Set `State` → `blocked`
- Set `Comment` → who or what is blocking
- Update `last_updated` to now
- Proceed to **Section 4: Pause Flow**

When a step does **not apply** in this run (wrong category, `qa-skill=none`, fast mode, operator opted out):
- Set `Completed` → `[~]`
- Set `State` → `skipped`
- Set `Comment` → reason (e.g., `fast mode`, `non-bug task`, `qa-skill=none`, `operator skipped`)
- Update `last_updated` to now

A `[~]` row is terminal. Resume skips over it the same as `[x]`.

### Comment Content Per Step

| Step | Comment |
|------|---------|
| 0. Dependency Check | `All present` or `Missing: <skill-list>` |
| 1. Get Task URL | `GID: <task-gid>` |
| 2. Fetch Task Details | `<task-id> — <task-name>` |
| 3. Validate Sprint-Readiness | `All checks passed` or `Fixed: <what was set via API>` |
| 4. Fetch Subtasks | `<N> subtasks (<M> complete, <K> remaining)` |
| 5. Fetch Comments & Attachments | `<N> comments, <M> attachments` |
| 6. Check Existing Work | `No existing branch` or `Resumed: <branch-name>` |
| 6b. Ask About Worktree | `worktree` or `current directory` |
| 6c. Confirm Base Branch | `<base-branch>` |
| 7. Create Feature Branch | `<branch-name> off <base>` |
| 8. Create Draft PR | `<pr-url>` |
| 9a. Move to In Progress | `Moved`, `Already in progress`, or `Failed: <reason>` |
| 9b. Post Start Comment | `Posted` or `Skipped (duplicate)` |
| 10. Route to Workflow | `fix-bug`, `brainstorm`, `feature-dev`, or `fast` |
| QA: Resolve | `web-qa`, `mobile-qa`, `none`, or skipped reason |
| QA: Investigate Bug | `Confirmed`, `Cannot reproduce`, or skipped reason |
| QA: Fix Bug | `Fix ready`, `Failed: <reason>`, or skipped reason |
| QA: Verify Fix | `Pass`, `Fail`, or skipped reason |
| QA: Verify Non-Bug | `Passed`, `Failed: <reason>`, or skipped reason |
| 12. Ship It | `Shipped: <pr-url>` |

### Frontmatter Updates

- After Step 2: set `task_id`
- After Step 7: set `branch` and `base_branch`
- After Step 10: set `workflow`

---

## Section 4: Pause Flow (Blocked)

### Trigger Phrases

- "park this", "I'm blocked", "pause task"
- "need to wait for an answer", "put this on hold"
- "waiting on [someone]", "blocked by [someone]"
- "save my progress", "pick this up later", "come back to this later"

### Steps

**1. Verify branch**
Confirm the current branch matches the task's branch (steps mode: `branch` in frontmatter). If on a different branch, warn before proceeding. If a merge is in progress, warn and do not commit.

**2. Commit WIP**
Stage all changes and commit with:
```
WIP: <task-id> — blocked on [short reason]
```
Example: `WIP: MT251-47 — blocked on timezone decision`

**3. Draft blocking question**
Formulate from conversation context. Present for user approval. The user MUST approve the exact wording before it is posted. Never post to Asana without explicit approval.

Example draft:

> "Hey @Sarah Chen — quick question before I can finish the CSV export: should timestamps use the user's local timezone or UTC? This affects how `formatTimestamp()` is implemented."

**4. Post to Asana**
After approval, post via the `asana-api` skill. Include the @mention of the blocking person.

**5. Update / save checkpoint**

- **Steps mode:** set the current step's `State` → `blocked`, `Comment` → who is blocking. Set frontmatter `paused_at` and `blocked_on`. Append a `## Notes` entry with the blocking question and who to follow up with.
- **Default mode:** create the checkpoint file now using the default-mode template. Fill in frontmatter, `## Blocked On`, `## Progress`, and `## Context for Resume`.

Run `mkdir -p .claude/checkpoints/` if the directory is missing. Ensure `.claude/checkpoints/` is in `.gitignore`.

**6. Push branch**
Push the WIP commit to remote.

**7. Confirm**
Report: commit hash, Asana comment link, checkpoint path. Instruct the user to run `/start-task` with the same URL to resume (add `steps` to keep steps-mode tracking if that's how it was started).

### Asana State

Leave the task status as "In Progress". Do not move the task to a different section.

---

## Section 5: Resume Flow

### Mode Detection

**Resume honors the checkpoint's original mode, not the current `$ARGUMENTS`.** A file created in steps mode always resumes as steps mode, even if the operator starts this session without `steps`; a default-mode file always resumes as default mode, even if the operator passes `steps`.

Detect the mode by reading the file:

- Contains a `## Steps` table → **steps mode**.
- Frontmatter + narrative body only → **default mode**.

If the operator's `$ARGUMENTS` mode disagrees with the file's mode, inform the operator briefly (`"Resuming as steps mode (checkpoint was created with checkpoints enabled)"`) and proceed using the file's mode. Do not ask — the file is authoritative.

### Entry Points

**Default mode:** resume is integrated into `start-task` Step 6a. After running `git fetch --prune`, check for a checkpoint file before performing branch detection.

**Steps mode:** resume is triggered by **Initialization (Section 2)** when a checkpoint file is found on entry — before any other step runs. Step 6a is skipped in steps mode.

### Trigger Phrases

- "resume task"
- "pick up where I left off"
- "continue [task-id]"
- "unpause"

### Steps

**1. Load checkpoint**

- Steps mode: find the first row where `Completed = [ ]`. That is where execution resumes. Skip all `[x]` and `[~]` rows entirely.
- Default mode: read frontmatter (`branch`, `workflow`, `paused_at`, `blocked_on`) and the narrative body.

**2. Present current state** — show the user a summary. In steps mode, render the Steps table. In default mode, show who it was blocked on and the progress notes.

**3. Check for blocked state** — if the resuming step has `State = blocked` (steps mode) or the checkpoint has `blocked_on` set (default mode), fetch Asana stories posted after `paused_at` / `last_updated`. Show new comments as potential answers. If no new comments: offer to resume anyway or keep waiting.

**4. Verify branch exists** — check the branch exists locally or on remote. If deleted: offer to recreate from `base_branch` or start fresh. If starting fresh, delete the checkpoint file.

**5. Check task status** — if the task has been completed, reassigned, or moved to a different section since `paused_at` / `last_updated`, warn before proceeding.

**6. Post resume comment** (only if previous state was blocked):

```
Resuming work on branch `<branch>`
```

**7. Resume work**

- Steps mode: check out the branch and continue from the first incomplete step.
- Default mode: check out the branch. Skip the validation steps that already ran before the pause. Load the checkpoint context into the working session. Route to the workflow specified in the `workflow` field.

**8. Clean up (default mode only)** — delete the checkpoint file after successful resume. The branch and Asana task are the source of truth going forward. In steps mode, the checkpoint persists until Step 12 completes (see Section 7).

---

## Section 6: Edge Cases

**Branch deleted since pause** — offer to recreate from `base_branch`, or start fresh (deletes checkpoint).

**Task completed in Asana since pause** — warn: "This task was completed in Asana on [date]. Resume anyway?"

**Task reassigned since pause** — warn: "This task is now assigned to [name]. Resume anyway?"

**Validation fails on resume at Step 3** — re-run validation fresh; present updated results.

**Step 9 fails on resume** — non-blocking; report and continue.

**No answer yet on resume** — if no new Asana comments have been posted since the pause, offer two options: resume anyway and continue without the answer, or keep waiting (exit without resuming).

---

## Section 7: Lifecycle End (Steps Mode)

After Step 12 (Ship It) completes successfully, **delete the checkpoint file**:

```bash
rm .claude/checkpoints/<task-gid>.md
```

The checkpoint's lifetime is one start → develop → ship pass. Once shipped, the Asana task status moves to "In Review" and the PR is ready for review — any further work on the branch (code-review fixes, reviewer feedback, follow-up commits) is **out of scope** for start-task and is not tracked by this checkpoint. It happens through git + the PR review thread, or via a different skill.

If the checkpoint is not deleted, a stale file may incorrectly trigger a resume the next time `/start-task` is run against the same task GID.

If Step 12 fails partway (e.g., `ship-it` sub-skill error), leave the checkpoint in place with `State → blocked` so the operator can retry. Only delete on successful completion of Step 12.

Default mode does not need a Section 7 equivalent — a default-mode checkpoint is already deleted by Resume Flow Step 8 on successful resume, so by the time Step 12 ships there is no file to clean up.
