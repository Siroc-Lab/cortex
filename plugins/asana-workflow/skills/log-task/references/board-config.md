# Board Config Reference

## Derive the project key

```bash
# Prefer git remote URL for stable, unique identity
git remote get-url origin 2>/dev/null \
  | sed 's|[^a-zA-Z0-9]|-|g' \
  | sed 's|-\{2,\}|-|g' \
  | tr '[:upper:]' '[:lower:]'
# Fallback if no remote:
basename "$(git rev-parse --show-toplevel 2>/dev/null || pwd)"
```

## Config file structure

```json
{
  "sprint_project_gid": "1234567890",
  "sprint_project_name": "ENG | Sprint 26.x",
  "backlog_project_gid": "9876543210",
  "backlog_project_name": "Product Backlog",
  "workspace_gid": "111111111"
}
```

## Resolve a project URL to name + workspace GID

Given a URL `app.asana.com/0/<gid>/...`, extract the GID with `app.asana.com/0/(\d+)`, then:

```bash
curl -s -H "Authorization: Bearer $ASANA_PERSONAL_ACCESS_TOKEN" \
  "https://app.asana.com/api/1.0/projects/<gid>?opt_fields=name,workspace.gid"
```

Use the returned `name` and `workspace.gid` to populate the config. Never ask the user for raw GIDs or workspace GIDs — always derive them from the URL.

## Save / update config

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
