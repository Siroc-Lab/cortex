---
name: log-task
description: >
  Creates an Asana task from work discovered or completed in the current conversation.
  Use when the user says things like "log this as a task", "create an Asana ticket for this",
  "capture this in Asana", "I want to track this", "let's create a ticket for what we just found",
  "add this to the backlog", "log this bug", "create a task before we start", or after discovering
  an issue together and wanting to formalize it before or after fixing it. Handles two variants:
  (1) plan-only — create task then hand off to start-task to begin work, and
  (2) fix-done — work is already complete, create task then hand off to ship-it via a clean
  git worktree. Always invoke this skill when the user's intent is clearly to create a new
  Asana task from the current conversation — even for short phrases like "quick asana ticket",
  "log this to asana", or "asana task for this". Do NOT trigger on generic logging requests
  ("log this error", "console.log", "log to Sentry") or on requests that reference an existing
  Asana task URL (those go to start-task or ship-it).
---

# Log Task

Take something discovered or completed in conversation and formalize it as an Asana task — then route to the right next step. This skill bridges unplanned work (bugs spotted, issues found, ad-hoc fixes) into the tracked development workflow.

## Prerequisites

- `$ASANA_PERSONAL_ACCESS_TOKEN` env var set — same as `start-task`. If missing, stop and guide setup.
- `asana-api` skill for all Asana API operations.
- `start-task` and `ship-it` skills available for routing.

---

## Step 1: Determine the Variant

Infer from conversation context which of the two paths applies. Do not ask unless genuinely ambiguous.

**Variant A — Plan Only**: Work has NOT been done yet. The user wants to create the Asana task first, then start working on it via `start-task`.
> Signals: "I want to plan this", "let me log it before we start", "we just figured out what needs to be done", task is still an idea.

**Variant B — Fix Done**: Work IS already done (or substantially done) in this session. The user wants to create the task retroactively and route to `ship-it`.
> Signals: "we just fixed it", "I already implemented this", "log what we just did", session has meaningful git changes or file edits.

State the inferred variant to the user before proceeding:
> "It looks like you've already fixed this — I'll create the task and hand off to ship-it. Is that right, or are you planning to start work after logging?"

---

## Step 2: Load Board Config

Config lives at `~/.claude/asana-workflow/<project-key>.json` — NOT in the repo. This keeps project-specific board GIDs out of source control while being available across sessions.

### Derive the project key

```bash
# Prefer git remote URL for stable, unique identity
git remote get-url origin 2>/dev/null \
  | sed 's|[^a-zA-Z0-9]|-|g' \
  | sed 's|-\{2,\}|-|g' \
  | tr '[:upper:]' '[:lower:]'
# Fallback if no remote:
basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
```

### Config file structure

```json
{
  "sprint_project_gid": "1234567890",
  "sprint_project_name": "ENG | Sprint 26.x",
  "backlog_project_gid": "9876543210",
  "backlog_project_name": "Product Backlog",
  "workspace_gid": "111111111"
}
```

### If config is missing — ask directly

```
No board config found for this project. I need two Asana project GIDs:

1. Sprint board — name + GID
   (Find the GID in the Asana URL: app.asana.com/0/<project-gid>/...)
2. Backlog board — name + GID

I'll save them to ~/.claude/asana-workflow/<project-key>.json (not in the repo).
```

---

## Step 3: Discover Custom Fields

Fetch the custom field definitions from the Sprint project. This is how you find out what fields exist and what values are valid — field names and enum options vary across projects, so never hardcode them.

```bash
curl -s -H "Authorization: Bearer $ASANA_TOKEN" \
  "https://app.asana.com/api/1.0/projects/<sprint_project_gid>/custom_field_settings\
?opt_fields=custom_field.gid,custom_field.name,custom_field.type,\
custom_field.enum_options,custom_field.enum_options.gid,custom_field.enum_options.name"
```

From the response, identify the fields you'll want to populate. Match by name using fuzzy, case-insensitive patterns:

| Intent | Match patterns |
|--------|---------------|
| Priority | "priority", "urgency", "severity" |
| Sizing | "size", "sizing", "story points", "points", "t-shirt" |
| Estimate | "estimate", "estimated time", "time estimate", "effort" |
| Product Status | "product status", "status", "state" |
| Assignee | handled natively — not a custom field |

For each matched field, record its GID and the full list of enum options (name + GID). You'll use these in Step 4 to pick sensible defaults and in Step 5 to show the user.

If a field has no match in the project, skip it gracefully — note it in the draft but don't block creation.

Also fetch the current user's GID for default assignment:

```bash
curl -s -H "Authorization: Bearer $ASANA_TOKEN" \
  "https://app.asana.com/api/1.0/users/me?opt_fields=gid,name,email"
```

---

## Step 4: Gather Task Details and Pick Defaults

Extract from conversation context as much as possible — do not ask for things already said. Fill any gaps with smart defaults:

| Field | Default | Notes |
|-------|---------|-------|
| **Title** | Summarize in ≤ 72 chars | From conversation; make it specific |
| **Description** | Summary of issue/fix/plan | Include root cause if known |
| **Priority** | Highest available option | e.g., P0, Critical, Urgent — pick the top enum option |
| **Sizing** | Lowest available option | e.g., XS, 1, S — if Variant B, use session scope as proxy |
| **Estimate** | Lowest available option | Same proxy logic as sizing |
| **Assignee** | Current user (Variant B) / Unassigned (Variant A) | |
| **Product Status** | "Assigned" enum option | Match case-insensitively |

**Sizing/Estimate for Variant B**: The work is done, so use it as a guide. Brief fix (< 1h) → smallest size. Moderate (1–4h) → second-smallest. Substantial (4h+) → medium. Err toward smaller — this task is being logged retroactively.

**Priority**: Default to the highest urgency enum option unless the user has indicated otherwise in conversation. If the conversation describes something non-critical ("nice to have", "minor cleanup"), drop to a mid-level.

---

## Step 5: Present Full Draft for Confirmation (REQUIRED)

Show everything — boards, fields, and values — before creating anything. The user must confirm. This step is non-negotiable.

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

If any field options couldn't be discovered (field not in project), show them as `— (not available in this project)` rather than omitting them.

If the user edits a field, update the draft and show it again. Do not create until explicit confirmation.

---

## Step 6: Create the Task

Create once confirmed. Use the `asana-api` skill patterns.

### 6a. Create the task

```bash
curl -s -X POST -H "Authorization: Bearer $ASANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "data": {
      "name": "<title>",
      "notes": "<description>",
      "workspace": "<workspace_gid>",
      "assignee": "<user_gid or null>",
      "custom_fields": {
        "<priority_gid>": "<selected_enum_gid>",
        "<sizing_gid>": "<selected_enum_gid>",
        "<estimate_gid>": "<selected_enum_gid>",
        "<product_status_gid>": "<assigned_enum_gid>"
      }
    }
  }' \
  "https://app.asana.com/api/1.0/tasks"
```

Only include custom fields that were successfully discovered. Save the returned `task_gid`.

For `assignee`: send `"assignee": null` explicitly (not omitting the field) for Variant A. Omitting it vs sending null behaves the same in the Asana API, but being explicit avoids ambiguity.

### 6b. Add to Sprint project

```bash
curl -s -X POST -H "Authorization: Bearer $ASANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"project":"<sprint_project_gid>"}}' \
  "https://app.asana.com/api/1.0/tasks/<task_gid>/addProject"
```

### 6c. Add to Backlog project

```bash
curl -s -X POST -H "Authorization: Bearer $ASANA_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"project":"<backlog_project_gid>"}}' \
  "https://app.asana.com/api/1.0/tasks/<task_gid>/addProject"
```

### 6d. Fetch the task ID field

After adding to projects, re-fetch the task to get the auto-assigned ID field (e.g., `MT251-182`). This is set by Asana automation once the task is in the right project:

```bash
curl -s -H "Authorization: Bearer $ASANA_TOKEN" \
  "https://app.asana.com/api/1.0/tasks/<task_gid>?opt_fields=custom_fields,custom_fields.name,custom_fields.display_value,custom_fields.type"
```

Look for a text-type custom field whose `display_value` matches the pattern `[A-Z]+-\d+` (uppercase prefix, hyphen, digits). Save this as `<task_id>`.

If the ID field is not yet set (automation can take a moment), retry once after a brief wait. If still absent, derive a sensible prefix from context:
- Bug fix → `fix/<slug>`
- Feature → `feat/<slug>`
- Tech debt / cleanup → `chore/<slug>`
- Documentation → `docs/<slug>`

### 6e. Report success

```
✓ Task created: Fix null pointer in export pipeline when CSV is empty
  ID:      MT251-182
  Asana:   https://app.asana.com/0/<sprint_project_gid>/<task_gid>
  Boards:  ENG | Sprint 26.x + Product Backlog
```

---

## Step 7: Route to the Next Step

### Variant A — Plan Only

Hand off to `start-task` with the new task URL:

> "Task logged. Ready to start work on it now? I'll invoke start-task."

Invoke `start-task` with `$ARGUMENTS = https://app.asana.com/0/<sprint_project_gid>/<task_gid>`.

`start-task` will handle branch creation, validation, and workflow routing from there.

---

### Variant B — Fix Done: Worktree + Ship

The work is done. The goal is to get those changes onto a clean branch without disrupting the current working directory or any other in-flight work.

#### 7b-1. Identify what changed

Capture the set of changed files relative to `main` (both staged and unstaged):

```bash
# Files changed vs main (uncommitted)
git diff --name-only main

# If there are commits on the current branch beyond main:
git diff --name-only main...HEAD
```

Save the union of both lists as `<changed_files>`.

#### 7b-2. Determine the branch name

Use the task ID from Step 6d:

```
<task_id>/<slug>
# e.g., MT251-182/fix-csv-export-null-crash
```

Where `<slug>` is the task title lowercased, spaces to hyphens, max 40 chars, alphanumeric and hyphens only.

If no task ID was resolved, use the type prefix (`fix/`, `feat/`, `chore/`, `docs/`) + slug.

#### 7b-3. Create a git worktree

Create a sibling worktree so the new branch lives in its own directory without touching the current workspace:

```bash
# Determine worktree path — sibling to the repo root
REPO_ROOT=$(git rev-parse --show-toplevel)
WORKTREE_PATH="${REPO_ROOT}/../$(basename $REPO_ROOT)-<task-id-slug>"

# Ensure main is up to date
git fetch origin main

# Create worktree on a new branch based on main
git worktree add "$WORKTREE_PATH" -b "<branch_name>" origin/main
```

Report the worktree path to the user:
> "Created worktree at `../cortex-MT251-182` on branch `MT251-182/fix-csv-export-null-crash`."

#### 7b-4. Copy changes into the worktree

Copy each changed file from the current working directory into the worktree:

```bash
for f in <changed_files>; do
  # Create directory if needed
  mkdir -p "$WORKTREE_PATH/$(dirname $f)"
  cp "$REPO_ROOT/$f" "$WORKTREE_PATH/$f"
done
```

Then stage and commit in the worktree:

```bash
cd "$WORKTREE_PATH"
git add <changed_files>
git commit -m "<task_id> :: <task-title-slug>"
```

#### 7b-5. Invoke ship-it in the worktree context

Tell the user:
> "Changes are on branch `MT251-182/fix-csv-export-null-crash` in `../cortex-MT251-182`. Handing off to ship-it."

Invoke `ship-it`. Thread the Asana task GID and URL so `ship-it` can skip re-asking for them:
- Task GID: `<task_gid>`
- Task URL: `https://app.asana.com/0/<sprint_project_gid>/<task_gid>`

`ship-it` should detect the task context from conversation and skip its Asana discovery step.

---

## Error Handling

- Never silently fail an API call — report status code and error.
- If task creation succeeds but adding to a project fails, do not roll back. Report the task URL so the user can add it manually.
- If the worktree path already exists, append a counter suffix (`-2`, `-3`) rather than failing.
- If `git worktree add` fails (e.g., branch already exists), report clearly and ask whether to reuse the existing branch or pick a new name.
- If custom field GIDs can't be resolved, create the task without those fields and list what's missing in the success report.
- If the task ID doesn't appear after two fetch attempts, proceed with the slug-based branch name and note that the ID may appear in Asana shortly.

---

## Config Save/Update

When saving a new or updated config, write it to disk and confirm to the user:

```bash
mkdir -p ~/.claude/asana-workflow
cat > ~/.claude/asana-workflow/<project-key>.json << 'CONF'
{
  "sprint_project_gid": "...",
  "sprint_project_name": "...",
  "backlog_project_gid": "...",
  "backlog_project_name": "...",
  "workspace_gid": "..."
}
CONF
```

> "Config saved to `~/.claude/asana-workflow/<project-key>.json` — not in the repo, won't be committed."
