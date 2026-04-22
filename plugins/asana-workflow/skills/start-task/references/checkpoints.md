# Checkpoints Reference

Checkpoints let `start-task` pause and resume work without losing context. There are two tracking modes, each with its own reference file:

- **Default mode (no `steps` flag):** a checkpoint is only written when the user pauses via the Pause Flow. Narrative body, no step table. Details: **`checkpoints-pause.md`**.
- **Steps mode (`$ARGUMENTS` contains `steps`):** the checkpoint is initialized on entry and updated around every step. Step table is the single source of truth for progress. Details: **`checkpoints-steps.md`**.

This file covers what's shared between the two modes: file location, frontmatter fields, mode detection on resume, the pause flow, the resume flow, and edge cases.

## Mode Map

| Concern | Default mode | Steps mode |
|---------|:---:|:---:|
| File created on | operator pause | skill entry (init) |
| File format | narrative body | steps table |
| Per-step updates | — | required |
| Pause flow | ✓ (creates file) | ✓ (updates existing row) |
| Resume flow | ✓ | ✓ |
| Cleanup | on successful resume | after Step 12 |
| Details | **`checkpoints-pause.md`** | **`checkpoints-steps.md`** |

---

## File Location

**Path:** `.claude/checkpoints/<task-gid>.md`

Use the Asana task GID (numeric) as the filename — available from the URL before any API call. Create the directory on first use: `mkdir -p .claude/checkpoints/`. Ensure `.claude/checkpoints/` is in `.gitignore` — these are local-only state files and must not be committed.

**Never overwrite an existing checkpoint.** If the file exists, work was already started — read it, determine the mode (below), and resume.

---

## Mode Detection (on Resume)

**Resume honors the checkpoint's original mode, not the current `$ARGUMENTS`.** A file created in steps mode always resumes as steps mode, even if the operator starts this session without `steps`; a default-mode file always resumes as default mode, even if the operator passes `steps`.

Detect the mode by reading the file:

- Contains a `## Steps` table → **steps mode** → follow `checkpoints-steps.md` for the file format and update rules.
- Frontmatter + narrative body only (no `## Steps` table) → **default mode** → follow `checkpoints-pause.md` for the file format.

If the operator's `$ARGUMENTS` mode disagrees with the file's mode, inform the operator briefly (`"Resuming as steps mode (checkpoint was created with checkpoints enabled)"`) and proceed using the file's mode. Do not ask — the file is authoritative.

---

## Shared Frontmatter Fields

Both mode templates use these core fields. Mode-specific files may add more.

| Field | Description |
|-------|-------------|
| `task_gid` | Asana task GID (numeric, from URL) |
| `task_id` | Project ID custom field (e.g., `MT251-47`) |
| `asana_url` | Full Asana task URL |
| `branch` | Feature branch name |
| `base_branch` | Branch the feature branch was created from |
| `workflow` | `fix-bug`, `brainstorm`, `feature-dev`, or `fast` |
| `paused_at` | ISO 8601 timestamp — set when entering the Pause Flow; used to filter new Asana comments on resume |
| `blocked_on` | Name of the person who should answer (informational) — set when blocked |

Steps mode adds `created_at` and `last_updated` (see `checkpoints-steps.md`).

---

## Pause Flow

### Trigger Phrases

- "park this", "I'm blocked", "pause task"
- "need to wait for an answer", "put this on hold"
- "waiting on [someone]", "blocked by [someone]"
- "save my progress", "pick this up later", "come back to this later"

### Steps

**1. Verify branch**
Confirm the current branch matches the task's branch. If on a different branch, warn before proceeding. If a merge is in progress, warn and do not commit.

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

**5. Save / update checkpoint** — **mode-specific**:

- **Steps mode:** see `checkpoints-steps.md` → "Pause Row Update".
- **Default mode:** see `checkpoints-pause.md` → "Pause File Creation".

**6. Push branch**
Push the WIP commit to remote.

**7. Confirm**
Report: commit hash, Asana comment link, checkpoint path. Instruct the user to run `/start-task` with the same URL to resume (add `steps` to keep steps-mode tracking if that's how it was started).

### Asana State

Leave the task status as "In Progress". Do not move the task to a different section.

---

## Resume Flow

### Entry Points

- **Default mode:** integrated into `start-task` Step 6a. After `git fetch --prune`, check for a checkpoint file before branch detection.
- **Steps mode:** triggered by **Initialization** (see `checkpoints-steps.md`) when a checkpoint is found on entry — before Step 0. Step 6a is skipped.

### Trigger Phrases

- "resume task"
- "pick up where I left off"
- "continue [task-id]"
- "unpause"

### Steps

**1. Load checkpoint and detect mode** — apply **Mode Detection** above. Then the mode-specific read:

- Steps mode: see `checkpoints-steps.md` → "Resume Row Scan" (find first `[ ]` row).
- Default mode: see `checkpoints-pause.md` → "Resume File Read" (read frontmatter + narrative body).

**2. Present current state** — show a summary. Steps mode: render the Steps table. Default mode: show `blocked_on` and progress notes.

**3. Check for blocked state** — if the resuming step has `State = blocked` (steps mode) or the checkpoint has `blocked_on` set (default mode), fetch Asana stories posted after `paused_at` / `last_updated`. Show new comments as potential answers. If no new comments: offer to resume anyway or keep waiting.

**4. Verify branch exists** — check the branch exists locally or on remote. If deleted: offer to recreate from `base_branch` or start fresh. Starting fresh deletes the checkpoint.

**5. Check task status** — if the task has been completed, reassigned, or moved to a different section since `paused_at` / `last_updated`, warn before proceeding.

**6. Post resume comment** (only if previous state was blocked):

```
Resuming work on branch `<branch>`
```

**7. Resume work** — **mode-specific**:

- Steps mode: check out the branch and continue from the first incomplete row.
- Default mode: check out the branch. Skip validation steps already run before the pause. Load the checkpoint context. Route to the workflow specified in `workflow`.

**8. Clean up** — **mode-specific**:

- **Default mode:** delete the checkpoint file after successful resume. Branch + Asana are the source of truth going forward.
- **Steps mode:** checkpoint persists until Step 12 completes. See `checkpoints-steps.md` → "Lifecycle End".

---

## Edge Cases

**Branch deleted since pause** — offer to recreate from `base_branch`, or start fresh (deletes checkpoint).

**Task completed in Asana since pause** — warn: "This task was completed in Asana on [date]. Resume anyway?"

**Task reassigned since pause** — warn: "This task is now assigned to [name]. Resume anyway?"

**Validation fails on resume at Step 3** — re-run validation fresh; present updated results.

**Step 9a or 9b fails on resume** — non-blocking; report and continue. 9a (move) failures leave the task in its prior section; 9b (post start comment) failures mean no 🏁 comment was posted.

**No answer yet on resume** — if no new Asana comments have been posted since the pause, offer two options: resume anyway and continue without the answer, or keep waiting (exit without resuming).
