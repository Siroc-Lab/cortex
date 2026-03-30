# Checkpoints Reference

## Section 1: File Format

**Location:** `.claude/checkpoints/<task-gid>.md`

Use the Asana task GID (numeric) as the filename — it is available from the URL before any API call is made.

Create the directory on first use: `mkdir -p .claude/checkpoints/`. Ensure `.claude/checkpoints/` is in `.gitignore` — these are local-only state files and must not be committed.

**Never overwrite an existing checkpoint on initialization.** If the file exists, work was already started — load it and resume.

### Frontmatter Fields

| Field | Description |
|-------|-------------|
| `task_gid` | Asana task GID (numeric, from URL) |
| `task_id` | Project ID custom field (e.g., `MT251-47`) — filled in after Step 2 |
| `asana_url` | Full Asana task URL |
| `branch` | Feature branch name — filled in after Step 7 |
| `base_branch` | Branch it was created from — filled in after Step 7 |
| `workflow` | `fix-bug`, `brainstorm`, or `feature-dev` — filled in after Step 11 |
| `created_at` | ISO 8601 timestamp — set once on initialization |
| `last_updated` | ISO 8601 timestamp — updated after every step |

---

### Steps Table

Six-column markdown table tracking every step of the workflow.

| Column | Values | Description |
|--------|--------|-------------|
| **Step** | Text | Step number and name |
| **Completed** | `[ ]` / `[x]` | Whether the step finished successfully |
| **Comment** | Text | Key data captured or what happened |
| **Attempts** | Integer | How many times this step has been attempted (starts at 0) |
| **State** | `—` / `in_progress` / `completed` / `blocked` | Current execution state |
| **Auto** | `[ ]` / `[x]` | `[x]` if completed without user input; `[ ]` if user approval was required |

---

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
| 9. Move to In Progress | [ ] | | 0 | — | [x] |
| 10. Post Start Comment | [ ] | | 0 | — | [x] |
| 11. Route to Workflow | [ ] | | 0 | — | [ ] |
| 12. Ship It | [ ] | | 0 | — | [x] |

## Notes
```

---

### Example (Partially Complete)

```markdown
---
task_gid: "1209384756102938"
task_id: MT251-47
asana_url: https://app.asana.com/0/123456789/1209384756102938
branch: MT251-47/add-csv-export
base_branch: main
workflow: ""
created_at: 2026-03-30T10:00:00Z
last_updated: 2026-03-30T10:14:32Z
---

## Steps

| Step | Completed | Comment | Attempts | State | Auto |
|------|-----------|---------|----------|-------|------|
| 1. Get Task URL | [x] | GID: 1209384756102938 | 1 | completed | [x] |
| 2. Fetch Task Details | [x] | MT251-47 — Add CSV export to dashboard | 1 | completed | [x] |
| 3. Validate Sprint-Readiness | [x] | Fixed: set Estimated time to 3h | 2 | completed | [ ] |
| 4. Fetch Subtasks | [x] | 3 subtasks (1 complete, 2 remaining) | 1 | completed | [x] |
| 5. Fetch Comments & Attachments | [x] | 2 comments, 1 screenshot attachment | 1 | completed | [x] |
| 6. Check Existing Work | [x] | No existing branch found | 1 | completed | [x] |
| 6b. Ask About Worktree | [x] | current directory | 1 | completed | [ ] |
| 6c. Confirm Base Branch | [x] | main | 1 | completed | [ ] |
| 7. Create Feature Branch | [x] | MT251-47/add-csv-export off main | 1 | completed | [x] |
| 8. Create Draft PR | [ ] | | 0 | in_progress | [x] |
| 9. Move to In Progress | [ ] | | 0 | — | [x] |
| 10. Post Start Comment | [ ] | | 0 | — | [x] |
| 11. Route to Workflow | [ ] | | 0 | — | [ ] |
| 12. Ship It | [ ] | | 0 | — | [x] |

## Notes
```

---

## Section 2: Initialization (Step 0)

**This is the very first action `start-task-steps` takes** — before any Asana API call, before any git command.

### Steps

1. **Extract task GID** from the URL in `$ARGUMENTS`. If no URL is provided, prompt for it now.

2. **Check for an existing checkpoint:**
   ```bash
   ls .claude/checkpoints/<task-gid>.md 2>/dev/null
   ```

3. **If checkpoint found** → this is a resume. Go to **Section 5: Resume Flow**.

4. **If no checkpoint** → create it now:
   ```bash
   mkdir -p .claude/checkpoints/
   ```
   Write the template with `task_gid`, `asana_url`, `created_at`, `last_updated` filled in. All steps: `[ ]`, state `—`, attempts `0`. Add `.claude/checkpoints/` to `.gitignore` if not already present.

---

## Section 3: Step Updates (Mandatory)

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

### Comment Content Per Step

| Step | Comment |
|------|---------|
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
| 9. Move to In Progress | `Moved` or `Already in progress` or `Failed: <reason>` |
| 10. Post Start Comment | `Posted` or `Skipped (duplicate)` |
| 11. Route to Workflow | `fix-bug`, `brainstorm`, or `feature-dev` |
| 12. Ship It | `Shipped: <pr-url>` |

### Frontmatter Updates

- After Step 2: set `task_id`
- After Step 7: set `branch` and `base_branch`
- After Step 11: set `workflow`

---

## Section 4: Pause Flow (Blocked)

### Trigger Phrases

- "park this", "I'm blocked", "pause task"
- "need to wait for an answer", "put this on hold"
- "waiting on [someone]", "blocked by [someone]"
- "save my progress", "pick this up later", "come back to this later"

### Steps

**1. Verify branch**
Confirm the current branch matches `branch` in checkpoint frontmatter. If on a different branch, warn before proceeding. If a merge is in progress, warn and do not commit.

**2. Commit WIP**
```
WIP: <task-id> — blocked on [short reason]
```

**3. Draft blocking question**
Formulate from conversation context. Present for user approval. Never post to Asana without explicit user approval.

**4. Post to Asana**
After approval, post via the `asana-api` skill.

**5. Update checkpoint**
Set the current step's `State` → `blocked`, `Comment` → who is blocking. Append a `## Notes` entry with the blocking question and who to follow up with.

**6. Push branch**
Push the WIP commit to remote.

**7. Confirm**
Report: commit hash, Asana comment link, checkpoint path. Instruct the user to run `start-task-steps` with the same URL to resume.

### Asana State
Leave the task as "In Progress". Do not move it.

---

## Section 5: Resume Flow

Resume is triggered by **Initialization (Section 2)** when a checkpoint file is found.

**1. Load checkpoint** — find the first row where `Completed = [ ]`. That is where execution resumes.

**2. Present current state** — show the Steps table so the user can see what's done and what remains.

**3. Check for blocked state** — if the resuming step has `State = blocked`, fetch Asana stories posted after `last_updated`. Show new comments as potential answers. If no new comments: offer to resume anyway or keep waiting.

**4. Verify branch** — check the branch exists locally or on remote. If deleted: offer to recreate from `base_branch` or start fresh.

**5. Check task status** — if the task has been completed or reassigned since `last_updated`, warn before proceeding.

**6. Post resume comment** (only if previous state was `blocked`):
```
Resuming work on branch `<branch>`
```

**7. Jump to the resuming step** — check out the branch and continue from the first incomplete step. All `[x]` rows are skipped entirely.

---

## Section 6: Edge Cases

**Branch deleted since pause** — offer to recreate from `base_branch`, or start fresh (deletes checkpoint).

**Task completed in Asana since pause** — warn: "This task was completed in Asana on [date]. Resume anyway?"

**Task reassigned since pause** — warn: "This task is now assigned to [name]. Resume anyway?"

**Validation fails on resume at Step 3** — re-run validation fresh; present updated results.

**Step 9 fails on resume** — non-blocking; report and continue.
