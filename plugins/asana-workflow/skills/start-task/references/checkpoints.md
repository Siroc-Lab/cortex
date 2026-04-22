# Checkpoints Reference

Every `start-task` run tracks progress in a checkpoint file so work can be paused and resumed without losing context. The checkpoint is initialized on entry, updated around every step, and deleted after a successful ship.

---

## File Location

**Path:** `.claude/checkpoints/<task-gid>.md`

Use the Asana task GID (numeric) as the filename — available from the URL before any API call. Create the directory on first use: `mkdir -p .claude/checkpoints/`. Ensure `.claude/checkpoints/` is in `.gitignore` — these are local-only state files and must not be committed.

**Never overwrite an existing checkpoint on initialization.** If the file exists, work was already started — load it and resume.

---

## File Format

A checkpoint has frontmatter, a `## Steps` table, and a `## Notes` section.

### Frontmatter Fields

| Field | Description |
|-------|-------------|
| `task_gid` | Asana task GID (numeric, from URL) |
| `task_id` | Project ID custom field (e.g., `MT251-47`) — filled in after Step 2 |
| `asana_url` | Full Asana task URL |
| `branch` | Feature branch name — filled in after Step 7 |
| `base_branch` | Branch the feature branch was created from — filled in after Step 7 |
| `workflow` | `fix-bug`, `brainstorm`, `feature-dev`, or `fast` — filled in after Step 10 |
| `created_at` | ISO 8601 timestamp — set once on initialization |
| `last_updated` | ISO 8601 timestamp — updated after every step |

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
| 6a. Ask About Worktree | [ ] | | 0 | — | [ ] |
| 6b. Confirm Base Branch | [ ] | | 0 | — | [ ] |
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

The `QA:` rows are only exercised when their preconditions apply (bug vs non-bug, resolved QA skill != `none`, not in fast mode). Prefer marking non-applicable rows `[~]` / `skipped` with a reason at the time they would have run, rather than leaving them as `[ ]` / `—`.

---

## Initialization

**This is the very first action `start-task` takes** — before any Asana API call, before any git command.

1. **Extract task GID** from the URL in `$ARGUMENTS`. If no URL is provided, prompt for it now.

2. **Check for an existing checkpoint:**
   ```bash
   ls .claude/checkpoints/<task-gid>.md 2>/dev/null
   ```

3. **If checkpoint found** → this is a resume. Go to **Resume Flow** below.

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

When a step is **blocked** (cannot complete — waiting on input, external dependency, or a failure that halts progress):
- Set `State` → `blocked`
- Set `Comment` → what is blocking
- Update `last_updated` to now
- Halt execution. The operator can investigate and resume later.

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
| 6a. Ask About Worktree | `worktree` or `current directory` |
| 6b. Confirm Base Branch | `<base-branch>` |
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

## Resume Flow

Triggered automatically during Initialization when a checkpoint file is found. Also triggered by explicit operator intent ("resume task", "pick up where I left off", "continue [task-id]", "unpause").

### Steps

**1. Load checkpoint** — find the first row where `Completed = [ ]`. That is where execution resumes. Skip all `[x]` and `[~]` rows entirely. If the target row has `State = blocked`, note what was blocking (from its `Comment`) and surface it to the operator before proceeding.

**2. Present current state** — render the Steps table so the operator sees what's done, what's remaining, and any blocked row.

**3. Verify branch exists** — check the branch exists locally or on remote. If deleted: offer to recreate from `base_branch` or start fresh. Starting fresh deletes the checkpoint.

**4. Check task status** — if the task has been completed, reassigned, or moved to a different section since `last_updated`, warn before proceeding.

**5. Resume work** — check out the branch and continue from the first incomplete row.

---

## Lifecycle End

After Step 12 (Ship It) completes successfully, **delete the checkpoint file**:

```bash
rm .claude/checkpoints/<task-gid>.md
```

The checkpoint's lifetime is one start → develop → ship pass. Once shipped, the Asana task status moves to "In Review" and the PR is ready for review — any further work on the branch (code-review fixes, reviewer feedback, follow-up commits) is **out of scope** for start-task and is not tracked by this checkpoint. It happens through git + the PR review thread, or via a different skill.

If the checkpoint is not deleted, a stale file may incorrectly trigger a resume the next time `/start-task` is run against the same task GID.

If Step 12 fails partway (e.g., `ship-it` sub-skill error), leave the checkpoint in place with `State → blocked` on the failing row so the operator can retry. Only delete on successful completion of Step 12.

---

## Edge Cases

**Branch deleted since the checkpoint was written** — offer to recreate from `base_branch`, or start fresh (deletes checkpoint).

**Task completed in Asana since last update** — warn: "This task was completed in Asana on [date]. Resume anyway?"

**Task reassigned since last update** — warn: "This task is now assigned to [name]. Resume anyway?"

**Validation fails on resume at Step 3** — re-run validation fresh; present updated results.

**Step 9a or 9b fails on resume** — non-blocking; report and continue. 9a (move) failures leave the task in its prior section; 9b (post start comment) failures mean no 🏁 comment was posted.
