# Checkpoints Reference — Steps Mode

Use this file when `steps_mode` is active. For shared concerns (file location, mode detection, pause flow, resume flow, edge cases) see **`checkpoints.md`**. For default (pause-only) mode see **`checkpoints-pause.md`**.

In steps mode, the checkpoint is initialized on skill entry and updated around every step. A step is not complete until its row is updated.

---

## File Format

A steps-mode checkpoint has full frontmatter, a `## Steps` table, and a `## Notes` section.

### Frontmatter Fields (steps mode adds these to the shared set)

| Field | Description |
|-------|-------------|
| `created_at` | ISO 8601 timestamp — set once on initialization |
| `last_updated` | ISO 8601 timestamp — updated after every step |

All other frontmatter fields (`task_gid`, `task_id`, `asana_url`, `branch`, `base_branch`, `workflow`, `paused_at`, `blocked_on`) are defined in `checkpoints.md` → Shared Frontmatter Fields. Some are filled in over time:

- `task_id` — after Step 2
- `branch` / `base_branch` — after Step 7
- `workflow` — after Step 10
- `paused_at` / `blocked_on` — on pause only

### Steps Table

Six-column markdown table tracking every step of the workflow.

| Column | Values | Description |
|--------|--------|-------------|
| **Step** | Text | Step number and name |
| **Completed** | `[ ]` / `[x]` / `[~]` | `[x]` done, `[~]` skipped as not-applicable, `[ ]` still open |
| **Comment** | Text | Key data captured or what happened |
| **Attempts** | Integer | How many times this step has been attempted (starts at 0) |
| **State** | `—` / `in_progress` / `completed` / `blocked` / `skipped` | Current execution state |
| **Auto** | `[ ]` / `[x]` | `[x]` if completed without user input; `[ ]` if user approval was required |

### Template (New Checkpoint)

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

The `QA:` rows are only exercised when their preconditions apply (bug vs non-bug, resolved QA skill != `none`, not in fast mode). Prefer marking non-applicable rows `[~]` / `skipped` with a reason at the time they would have run (see Step Updates → Update Pattern below), rather than leaving them as `[ ]` / `—`.

---

## Initialization

**This is the very first action `start-task` takes when `steps_mode` is active** — before any Asana API call, before any git command.

1. **Extract task GID** from the URL in `$ARGUMENTS`. If no URL is provided, prompt for it now.

2. **Check for an existing checkpoint:**
   ```bash
   ls .claude/checkpoints/<task-gid>.md 2>/dev/null
   ```

3. **If checkpoint found** → this is a resume. Apply **Mode Detection** from `checkpoints.md`. If the file is default-mode, resume proceeds as default mode even though the operator passed `steps` — the file's mode is authoritative. Follow the Resume Flow in `checkpoints.md`.

4. **If no checkpoint** → create it now:
   ```bash
   mkdir -p .claude/checkpoints/
   ```
   Write the template above with `task_gid`, `asana_url`, `created_at`, `last_updated` filled in. All steps: `[ ]`, state `—`, attempts `0`. Add `.claude/checkpoints/` to `.gitignore` if not already present.

---

## Step Updates

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
- Proceed to the **Pause Flow** in `checkpoints.md`. See also "Pause Row Update" below for the steps-mode-specific fill-in at Pause Flow step 5.

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

## Pause Row Update

When entering the Pause Flow (see `checkpoints.md`) in steps mode, step 5 of that flow ("Save / update checkpoint") maps to the following in the existing file:

- Set the current row: `State` → `blocked`, `Comment` → who/what is blocking.
- Set frontmatter: `paused_at` → now, `blocked_on` → person's name (if known).
- Append a `## Notes` entry with the blocking question and who to follow up with.

The file already exists (it was created at Initialization). Do not create a new file.

---

## Resume Row Scan

When Resume Flow (see `checkpoints.md`) step 1 ("Load checkpoint") runs in steps mode:

- Find the first row where `Completed = [ ]`. That is where execution resumes.
- Skip all `[x]` and `[~]` rows entirely — they are terminal.
- If the row's `State` is `blocked`, follow Resume Flow step 3 (check for Asana answers).

---

## Lifecycle End

After Step 12 (Ship It) completes successfully, **delete the checkpoint file**:

```bash
rm .claude/checkpoints/<task-gid>.md
```

The checkpoint's lifetime is one start → develop → ship pass. Once shipped, the Asana task status moves to "In Review" and the PR is ready for review — any further work on the branch (code-review fixes, reviewer feedback, follow-up commits) is **out of scope** for start-task and is not tracked by this checkpoint. It happens through git + the PR review thread, or via a different skill.

If the checkpoint is not deleted, a stale file may incorrectly trigger a resume the next time `/start-task` is run against the same task GID.

If Step 12 fails partway (e.g., `ship-it` sub-skill error), leave the checkpoint in place with `State → blocked` so the operator can retry. Only delete on successful completion of Step 12.

Default mode does not need this — a default-mode checkpoint is deleted by the Resume Flow on successful resume (see `checkpoints.md`), so by ship time there is no file to clean up.
