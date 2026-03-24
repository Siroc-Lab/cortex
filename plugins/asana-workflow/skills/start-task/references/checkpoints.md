# Checkpoints Reference

## Section 1: Checkpoint File Format

**Location:** `.claude/checkpoints/<task-id>.md`

Create the directory on first use with `mkdir -p .claude/checkpoints/`. Add `.claude/checkpoints/` to `.gitignore` — these are local-only state files and must not be committed.

If a checkpoint already exists for this task, overwrite it. Re-pausing replaces the previous checkpoint.

### Required Frontmatter Fields

| Field | Description |
|---|---|
| `task_id` | Project ID from Asana custom field (e.g., `MT251-47`) |
| `task_gid` | Asana task GID (numeric string) |
| `asana_url` | Full Asana task URL |
| `branch` | Feature branch name (e.g., `MT251-47/add-csv-export`) |
| `base_branch` | Branch it was created from (e.g., `main`) |
| `workflow` | Downstream skill: `feature-dev` or `systematic-debugging` |
| `phase` | Where pause happened: `discovery`, `implementation`, or `review` |
| `paused_at` | ISO 8601 timestamp — used to filter new Asana comments on resume |
| `blocked_on` | Name of the person who should answer (informational) |

### Example Checkpoint File

```markdown
---
task_id: MT251-47
task_gid: "1209384756102938"
asana_url: https://app.asana.com/0/123456789/1209384756102938
branch: MT251-47/add-csv-export
base_branch: main
workflow: feature-dev
phase: implementation
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

## Section 2: Pause Flow

### Trigger Phrases

Initiate the pause flow when the user says any of the following:
- "park this"
- "I'm blocked"
- "pause task"
- "need to wait for an answer"
- "put this on hold"
- "waiting on [someone]"
- "blocked by [someone]"
- "save my progress"
- "pick this up later"
- "come back to this later"

### Steps

**1. Verify branch**

Confirm the current branch matches the task's branch. If on a different branch, warn the user before proceeding. If a merge is in progress, warn the user and do not commit.

**2. Commit WIP**

Stage all changes and commit with the message format:
```
WIP: <task-id> — blocked on [short reason]
```

Example: `WIP: MT251-47 — blocked on timezone decision`

**3. Draft blocking question**

Formulate the blocking question from conversation context. Present it to the user for approval before posting anywhere. Example draft:

> "Hey @Sarah Chen — quick question before I can finish the CSV export: should timestamps use the user's local timezone or UTC? This affects how `formatTimestamp()` is implemented."

The user MUST approve the exact wording before it is posted. Never post to Asana without explicit approval.

**4. Post to Asana**

After user approval, post the comment via the `asana-api` skill. Include the @mention of the blocking person.

**5. Save checkpoint**

Write the checkpoint file to `.claude/checkpoints/<task-id>.md`. Run `mkdir -p .claude/checkpoints/` if needed. Ensure `.claude/checkpoints/` is in `.gitignore`.

**6. Push branch**

Push the WIP commit to the remote branch.

**7. Confirm**

Report what was done: commit hash, Asana comment link, checkpoint path. Tell the user to run `/start-task` with the same Asana URL to resume.

### Asana State

Leave the task status as "In Progress". Do not move the task to a different section.

---

## Section 3: Resume Flow

Resume is integrated into `start-task` Step 6. After running `git fetch --prune`, check for a checkpoint file before performing branch detection.

### Trigger Phrases

Initiate the resume flow when the user says:
- "resume task"
- "pick up where I left off"
- "continue [task-id]"
- "unpause"

### Steps

**1. Check for checkpoint file**

Look for `.claude/checkpoints/<task-id>.md`. If not found, fall through to the existing branch detection logic.

**2. Verify branch exists**

Check whether the branch exists locally or on the remote. If the branch has been deleted:
- Offer to recreate it from `base_branch`
- Or offer to start fresh

If starting fresh, delete the checkpoint file.

**3. Check Asana task status**

Fetch the current task state. If the task has been completed, reassigned, or moved to a different section since `paused_at`, warn the user before proceeding.

**4. Present checkpoint state**

Show the user a summary: branch name, who it was blocked on, and the progress notes from the checkpoint body.

**5. Check for answers**

Fetch Asana stories (comments) created after the `paused_at` timestamp. Filter for comment-type stories. Present any new comments to the user.

**6. Handle no answer**

If no new comments have been posted since the pause, offer two options:
- Resume anyway and continue without the answer
- Keep waiting (exit without resuming)

**7. Resume work**

Check out the branch. Skip the validation steps that already ran before the pause. Load the checkpoint context into the working session. Route to the workflow specified in the `workflow` field (`feature-dev` or `systematic-debugging`), starting at the `phase` where work was paused.

**8. Post resume comment**

Post a comment on the Asana task:
```
Resuming work on branch `<task-id>/<slug>`
```

**9. Clean up**

Delete the checkpoint file after successful resume. The branch and Asana task are the source of truth going forward.
