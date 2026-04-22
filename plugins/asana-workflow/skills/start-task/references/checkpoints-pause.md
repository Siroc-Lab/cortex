# Checkpoints Reference — Default Mode (Pause Only)

Use this file for default-mode checkpoints (i.e., when `steps_mode` is NOT active). For shared concerns (file location, mode detection, pause flow, resume flow, edge cases) see **`checkpoints.md`**. For steps-mode checkpoints see **`checkpoints-steps.md`**.

In default mode, a checkpoint is only created when the user triggers the Pause Flow. There is no initialization on entry, no per-step tracking, no lifecycle-end cleanup step. The file captures just enough context to resume paused work later.

---

## File Format

A default-mode checkpoint has frontmatter and a narrative body. No `## Steps` table.

### Frontmatter Fields

All fields come from the shared set in `checkpoints.md` → Shared Frontmatter Fields. At pause time, fill in:

- `task_gid`, `task_id`, `asana_url`, `branch`, `base_branch`, `workflow` — from the session state (whatever is known at pause)
- `paused_at` — the pause timestamp (now)
- `blocked_on` — name of the person blocking (if any)

`created_at` and `last_updated` (used by steps mode) are not set in default mode.

### Narrative Body

Three sections of plain markdown:

- `## Blocked On` — what you're waiting on, in prose
- `## Progress` — what's been completed so far
- `## Context for Resume` — anything needed to pick up the work (file paths, stubs, TODOs, etc.)

### Example

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

## Pause File Creation

When the Pause Flow (see `checkpoints.md`) reaches step 5 ("Save / update checkpoint") in default mode:

1. Run `mkdir -p .claude/checkpoints/` if the directory is missing. Ensure `.claude/checkpoints/` is in `.gitignore`.
2. Create `.claude/checkpoints/<task-gid>.md` using the template above (the file did not exist before this moment — default mode does not pre-create it).
3. Fill in frontmatter from session state: `task_gid`, `task_id`, `asana_url`, `branch`, `base_branch`, `workflow`, `paused_at` (now), `blocked_on` (if known).
4. Write `## Blocked On`, `## Progress`, and `## Context for Resume` sections based on the current conversation.

---

## Resume File Read

When Resume Flow (see `checkpoints.md`) step 1 ("Load checkpoint") runs in default mode:

- Read frontmatter (`branch`, `workflow`, `paused_at`, `blocked_on`) and the narrative body.
- Use the body's `## Context for Resume` section to re-establish working context in the new session.
- If `blocked_on` is set, follow Resume Flow step 3 (check for Asana answers since `paused_at`).

---

## Cleanup on Resume

Default-mode checkpoints are deleted after a successful resume — see `checkpoints.md` → Resume Flow, Step 8. The branch and Asana task are the source of truth going forward.

There is no Lifecycle End section for default mode — by the time Step 12 (Ship It) runs, the checkpoint has already been cleaned up on resume. If no pause ever happened, no file was created in the first place, so there is nothing to delete at ship time.
