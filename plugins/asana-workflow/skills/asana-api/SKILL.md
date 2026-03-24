---
name: asana-api
description: >
  Operate with Asana API - create, read, update tasks, projects, users, and all Asana resources
  using the node-asana SDK or direct REST calls. Use when the user mentions Asana tasks, projects,
  workspaces, or any Asana operations.
---

# Asana API

Common Asana REST API patterns for task management workflows. All operations use bearer token authentication via `$PERSONAL_ACCESS_TOKEN`.

## Prerequisites

- `$PERSONAL_ACCESS_TOKEN` env var — Asana personal access token
  - If missing: guide user to https://app.asana.com/0/my-apps
  - Add to `~/.zshrc`: `export PERSONAL_ACCESS_TOKEN="your-token"`

## Authentication

All requests include:

```
Authorization: Bearer $PERSONAL_ACCESS_TOKEN
```

## Common Operations

### Fetch Task Details

```bash
curl -s -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
  "https://app.asana.com/api/1.0/tasks/<task-gid>?opt_fields=name,notes,assignee,assignee.name,custom_fields,custom_fields.name,custom_fields.display_value,custom_fields.enum_value,custom_fields.enum_value.name,custom_fields.type,memberships,memberships.project,memberships.project.name,memberships.section,memberships.section.name,projects,projects.name"
```

### Move Task to Section

Move tasks between board columns (e.g., "In Progress", "In Review"):

1. List sections in the project:
   ```bash
   curl -s -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
     "https://app.asana.com/api/1.0/projects/<project-gid>/sections?opt_fields=name"
   ```

2. Find the target section by name, then move:
   ```bash
   curl -s -X POST -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"data":{"task":"<task-gid>"}}' \
     "https://app.asana.com/api/1.0/sections/<section-gid>/addTask"
   ```

### Update Custom Field

```bash
curl -s -X PUT -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"custom_fields":{"<field-gid>":"<value>"}}}' \
  "https://app.asana.com/api/1.0/tasks/<task-gid>"
```

For enum fields, the value is the enum option GID.

### Post Comment on Task

```bash
curl -s -X POST -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"data":{"text":"<comment text>"}}' \
  "https://app.asana.com/api/1.0/tasks/<task-gid>/stories"
```

### Fetch Subtasks

```bash
curl -s -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
  "https://app.asana.com/api/1.0/tasks/<task-gid>/subtasks?opt_fields=name,completed,gid"
```

### Fetch Task Stories (Comments)

```bash
curl -s -H "Authorization: Bearer $PERSONAL_ACCESS_TOKEN" \
  "https://app.asana.com/api/1.0/tasks/<task-gid>/stories?opt_fields=type,text,created_by.name,created_at"
```

Filter results for `type: "comment"` to get human-written comments.

## URL Formats and GID Extraction

Asana URLs come in several formats. The task GID is always a numeric segment:

- `https://app.asana.com/0/<project-gid>/<task-gid>`
- `https://app.asana.com/0/<project-gid>/<task-gid>/f`
- `https://app.asana.com/1/<org-gid>/project/<project-gid>/task/<task-gid>`
- `https://app.asana.com/1/<org-gid>/inbox/<inbox-gid>/item/<task-gid>/...`

## Error Handling

- **401 Unauthorized** — Token expired or invalid. Guide user to regenerate.
- **403 Forbidden** — User lacks access to the resource.
- **404 Not Found** — Invalid GID or deleted resource.
- **429 Rate Limited** — Back off and retry after the `Retry-After` header.

Never silently skip a failed API call. Report the status code and error message.

## Full API Reference

For the complete list of 232 endpoints across 45 resources, consult **`references/spec-summary.md`**.
