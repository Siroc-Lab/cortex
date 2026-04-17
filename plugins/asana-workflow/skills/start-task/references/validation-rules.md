# Sprint-Readiness Validation Rules

Four checks determine whether a task is ready to start. Track which pass and which fail, then present results as a checklist.

## Check 1: Active Sprint Membership

Before running checks, load the board registry cache following `board-resolution.md` (at the plugin root: `plugins/asana-workflow/references/board-resolution.md`). If the cache doesn't exist, run the Full Discovery flow. If `active_sprint.due_on` is in the past, run Sprint Auto-Refresh.

The task must be a member of the **active sprint** — compare the task's `memberships[].project.gid` values against `active_sprint.gid` from the cache.

A task that is only in an old/completed sprint does **not** pass this check — it must be pulled into the current active sprint in Asana first.

**This check is non-negotiable.** If the task is not in the active sprint, it cannot be started — there is no skip for this.

## Check 2: Estimated Time

The custom field named "Estimated time" must have a non-null `display_value`.

If missing, offer to set it via the API:

> No estimated time set — how long do you estimate?

## Check 3: Product Status = Assigned

The custom field named "Product Status" must have an `enum_value.name` of "Assigned".

If incorrect, offer to update it via the API.

## Check 4: ID Field

There must be a text-type custom field whose `display_value` matches an ID pattern like `XXX-123` (uppercase letters, a hyphen, then digits). The field name varies per project (e.g., "MT251", "BI", "PD253") — match by pattern, not by field name.

## Failure Display Format

Separate blocking failures from skippable ones. Present them distinctly:

**If any blocking check fails (Active sprint membership, Estimated time, Product Status):**
```
Sprint-Readiness Checks:
- [x] Active sprint membership — ENG | Sprint 26.16
- [ ] Estimated time — Not set (REQUIRED)
- [ ] Product Status — "Ready" (needs "Assigned") (REQUIRED)
- [x] Has ID: MT251-12

Two required fields are missing. Want me to set them via API?
```

For active sprint membership: the task must be added to the current sprint (ENG | Sprint 26.16) in Asana manually — cannot be set via API.
For Estimated time: prompt for the estimate, then set via API.
For Product Status: offer to set to "Assigned" via API.

Do not proceed until all three blocking checks pass.

**If only the ID field fails (skippable):**
```
Sprint-Readiness Checks:
- [x] Active sprint membership — ENG | Sprint 26.16
- [x] Estimated time — 3h
- [x] Product Status — Assigned
- [ ] ID field — Not set

ID is missing. Skip and continue, or set it in Asana first?
```

Warn once about traceability risks, then proceed if skipped.

## Skip Rules

**Active sprint membership, Estimated time, Product Status** — never skippable. Block the workflow. Offer to set Estimated time and Product Status via API, but do not proceed until all three are resolved.

**ID field** — if the intent is to skip ("just start", "I'll fix Asana later"), warn once about traceability risks, then proceed.
