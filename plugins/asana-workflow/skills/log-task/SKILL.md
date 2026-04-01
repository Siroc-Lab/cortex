---
name: log-task
description: >
  Use when the user wants to create a new Asana task from the current conversation — before
  or after doing the work. Triggers: "log this as a task", "create an Asana ticket for this",
  "capture this in Asana", "I want to track this", "let's create a ticket for what we just found",
  "add this to the backlog", "log this bug", "create a task before we start", "quick asana ticket",
  "log this to asana", "asana task for this", or after discovering and fixing an issue together.
  Do NOT trigger on generic logging requests ("log this error", "console.log", "log to Sentry")
  or on requests that reference an existing Asana task URL (those go to start-task or ship-it).
argument-hint: "[sprint: <sprint-url>] [backlog: <backlog-url>]"
---

# Log Task

Take something discovered or completed in conversation and formalize it as an Asana task — then route to the right next step. This skill bridges unplanned work (bugs spotted, issues found, ad-hoc fixes) into the tracked development workflow.

## Prerequisites

- `asana-api` skill for all Asana API operations — route all Asana API calls through it, no raw curl.
- `start-task` and `ship-it` skills available for routing.

---

## Step 0: Parse Arguments for URL Overrides

Before doing anything else, scan `$ARGUMENTS` and the triggering message for Asana project URLs.

Extract GIDs from URLs using the pattern `app.asana.com/0/(\d+)` — the GID is the first path segment after `/0/`.

**Disambiguate sprint vs backlog** using context words near each URL:

| Trigger words (case-insensitive) | Detected as |
|---|---|
| "sprint", "current sprint", "new sprint", "use sprint" | sprint override |
| "backlog", "product backlog", "use backlog", "new backlog" | backlog override |
| No context words | ambiguous |

For each URL detected, resolve the project name and workspace GID via the API (see `references/board-config.md`). Store as `sprint_override` and/or `backlog_override` (each with `gid` + `name`), and `workspace_override` if returned.

**Ambiguity rules:**
- Two URLs with no context → ask once: "Which is the sprint board and which is the backlog?"
- One URL with no context → ask: "Is this the sprint board or the backlog?"
- URL that isn't a valid Asana project URL → ignore silently

Carry resolved overrides into Step 2.

---

## Step 1: Determine the Variant

Infer from conversation context which of the two paths applies. Proceed silently — do not announce the inferred variant or ask for confirmation unless the signals are genuinely ambiguous.

**Plan Only**: Work has NOT been done yet. The user wants to create the Asana task first, then optionally start working on it.
> Signals: "I want to plan this", "let me log it before we start", "we just figured out what needs to be done", task is still an idea.

**Fix Done**: Work IS already done (or substantially done) in this session. The user wants to create the task retroactively.
> Signals: "we just fixed it", "I already implemented this", "log what we just did", session has meaningful git changes or file edits.

When confident, announce the inferred variant and proceed immediately — do not wait:
- Plan Only: > "Treating this as a plan-only task — I'll create the Asana task and ask if you want to start work after."
- Fix Done: > "Treating this as a completed fix — I'll create the Asana task and set up a branch for shipping."

**If genuinely ambiguous** (no clear signals either way), ask and wait:
> "Is the work already done, or are you planning to start after logging? [done/plan]"

---

## Step 2: Load Board Config

Config lives at `~/.claude/asana-workflow/<project-key>.json` — NOT in the repo.

Read `references/board-config.md` for the project key derivation command, config file structure, and save/update template.

### Apply overrides from Step 0

After loading (or failing to load) config, merge any overrides:

- If `sprint_override` is set → replace `sprint_project_gid` and `sprint_project_name`
- If `backlog_override` is set → replace `backlog_project_gid` and `backlog_project_name`
- If `workspace_override` is set and config has no `workspace_gid` → set it

If any override was applied, save the updated config and report:
```
Sprint board updated: ENG | Sprint 27.x  (was: ENG | Sprint 26.x)
Config saved to ~/.claude/asana-workflow/<project-key>.json
```

### If config is missing — ask for URLs

If config is missing AND no overrides were provided for a board, ask:

```
No board config found for this project. I need the Asana URLs for:

1. Sprint board — paste the URL from your browser (app.asana.com/0/<project-gid>/...)
2. Backlog board — paste the URL from your browser

I'll save them to ~/.claude/asana-workflow/<project-key>.json (not in the repo).
```

If overrides cover one board but not the other, ask only for the missing one. Once URLs are provided, derive GIDs and workspace GID via the API (see `references/board-config.md`). Never ask the user for raw GIDs.

---

## Step 3: Discover Custom Fields

Read `references/asana-api-calls.md` for the custom field discovery call and the current user fetch.

Match fields by name using fuzzy, case-insensitive patterns:

| Intent | Match patterns |
|--------|---------------|
| Priority | "priority", "urgency", "severity" |
| Sizing | "size", "sizing", "story points", "points", "t-shirt" |
| Estimate | "estimate", "estimated time", "time estimate", "effort" |
| Product Status | "product status", "status", "state" |
| Assignee | handled natively — not a custom field |

Record each matched field's GID and full enum options list. If a field has no match, skip it gracefully.

---

## Step 4: Gather Task Details and Pick Defaults

Extract from conversation context as much as possible. Fill gaps with smart defaults:

| Field | Default | Notes |
|-------|---------|-------|
| **Title** | Summarize in ≤ 72 chars | From conversation; make it specific |
| **Description** | Summary of issue/fix/plan | Include root cause if known |
| **Priority** | Highest urgency option | Match semantically: prefer names containing "0", "critical", or "urgent"; or the option with the lowest numeric suffix (P0 < P1 < P2). Fall back to first in list only if no semantic match. |
| **Sizing** | Lowest available option | e.g., XS, 1, S — if Fix Done, use session scope as proxy |
| **Estimate** | Lowest available option | Same proxy logic as sizing |
| **Assignee** | Current user (Fix Done) / Unassigned (Plan Only) | |
| **Product Status** | "Assigned" enum option | Match case-insensitively |

**Sizing/Estimate for Fix Done**: Brief fix (< 1h) → smallest. Moderate (1–4h) → second-smallest. Substantial (4h+) → medium. Err toward smaller.

**Priority**: Default to the highest urgency enum option using semantic matching — do not rely on list position. If the conversation describes something non-critical ("nice to have", "minor cleanup"), drop to a mid-level option instead.

---

## Step 5: Present Full Draft for Confirmation (REQUIRED)

Show everything before creating anything. The user must confirm. This step is non-negotiable.

```
Task draft:
  Title:          Fix null pointer in export pipeline when CSV is empty
  Description:    The CSV exporter crashes when the input DataFrame has zero
                  rows. Root cause: missing empty-check before column iteration.

  Boards:
    Sprint:  ENG | Sprint 26.x  (GID: 1234567890)
    Backlog: Product Backlog    (GID: 9876543210)

  Fields:
    Priority:       P0            [options: P0, P1, P2, P3]
    Sizing:         XS            [options: XS, S, M, L, XL]
    Estimate:       30m           [options: 30m, 1h, 2h, 4h, 1d]
    Product Status: Assigned      [options: Assigned, In Progress, Done]
    Assignee:       Francisco Javier (you)

  Fields not found in this project: (none)

Create this task? [Y/n / type field name to edit]
```

If a field couldn't be discovered, show it as `— (not available in this project)`. If the user edits a field, update the draft and show it again. Do not create until explicit confirmation.

---

## Step 6: Create the Task

Read `references/asana-api-calls.md` for all curl commands. Execute in this order — do not skip or reorder:

1. **6a** — Create the task (workspace + assignee only, no custom fields)
2. **6b** — Add to Sprint project
3. **6c** — Add to Backlog project
4. **6d** — Set custom fields via PUT (now that the task is in a project)
5. **6d cont** — Fetch the auto-assigned task ID (retry up to 3 times, 10s apart). If still absent after all retries, derive a prefix from context:
   - Bug fix → `fix/<slug>`, Feature → `feat/<slug>`, Tech debt → `chore/<slug>`, Docs → `docs/<slug>`

Report success:
```
✓ Task created: Fix null pointer in export pipeline when CSV is empty
  ID:      MT251-182
  Asana:   https://app.asana.com/0/<sprint_project_gid>/<task_gid>
  Boards:  ENG | Sprint 26.x + Product Backlog
```

**Error handling:**
- Never silently fail an API call — report status code and error.
- If task creation succeeds but adding to a project fails, do not roll back. Report the task URL so the user can add it manually.
- If custom field GIDs can't be resolved, create the task without those fields and list what's missing.

---

## Step 7: Route to the Next Step

### Plan Only

Ask the user whether to proceed:

> "Task logged. Want to start work on it now, or leave it for later? [start/later]"

**Wait for the user's response.**

- If start → invoke `start-task` with `$ARGUMENTS = https://app.asana.com/0/<sprint_project_gid>/<task_gid>`.
- If later (or any non-committal response) → stop here. The user can start later with `/start-task <url>`.

---

### Fix Done: Worktree + Ship

Read `references/worktree-flow.md` for all git commands.

#### 7b-1. Identify what changed
Run the git diff commands from `references/worktree-flow.md`. If `<changed_files>` is empty, stop and ask the user where the changes are before continuing.

#### 7b-2. Ask before touching git

> "Task logged. Want to create a branch and ship now, or leave it for later? [ship/later]"

**Wait for the user's response.**

- If later (or any non-committal response) → stop here. The user can run `/ship-it` whenever ready.
- If ship → continue to 7b-3.

#### 7b-3. Determine the branch name

```
<task_id>/<slug>
# e.g., MT251-182/fix-csv-export-null-crash
```

`<slug>` = task title lowercased, spaces to hyphens, max 40 chars, alphanumeric and hyphens only. If no task ID was resolved, use the type prefix (`fix/`, `feat/`, `chore/`, `docs/`) + slug.

#### 7b-4 & 7b-5. Create worktree and copy changes
Follow the worktree creation and file copy steps in `references/worktree-flow.md`.

#### 7b-6. Invoke ship-it

Thread the Asana task GID and URL so `ship-it` can skip re-asking:
- Task GID: `<task_gid>`
- Task URL: `https://app.asana.com/0/<sprint_project_gid>/<task_gid>`

---

## Step 8: Harvest Reminder

> "Don't forget to move any Harvest time logged for this work to **<task_id>: <task_title>**."

Do not attempt to interact with Harvest directly — this step is advisory only.
