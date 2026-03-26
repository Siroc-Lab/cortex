# SIROC Cortex

Central repository for SIROC's AI context: skills, agents, hooks, and orchestration logic. Distributed as a [Claude Code plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces).

## Marketplace

**Name:** `siroc-cortex`

## Plugins

### asana-workflow (v1.0.0)

End-to-end Asana-driven development workflow: from ticket to shipped PR with automated task tracking, git management, and team communication.

**Skills included:**

| Skill | Description |
|-------|-------------|
| `start-task` | Validates Asana task, creates branch and draft PR, routes to feature-dev or debugging |
| `ship-it` | Orchestrates pre-checks, summary, PR creation, and Asana update |
| `pre-ship-check` | Validates git state, lint, build, and tests |
| `git-check` | Branch safety, working tree cleanliness, debug artifact detection |
| `work-summary` | Session recap for standups, handoffs, and PRs |
| `create-pr` | Full PR lifecycle with Asana linking and reviewer assignment |
| `asana-api` | Asana REST API patterns and common operations |

## Installation

See [INSTALL.md](INSTALL.md) for setup instructions.

## Repository Structure

```
.claude-plugin/
  marketplace.json        # Marketplace catalog
plugins/
  asana-workflow/
    .claude-plugin/
      plugin.json         # Plugin manifest
    skills/
      start-task/         # Entry point skill + references
      ship-it/            # Shipping orchestrator
      pre-ship-check/     # Readiness gate
      git-check/          # Git state validation
      work-summary/       # Session recap
      create-pr/          # PR lifecycle
      asana-api/          # Asana API operations + spec reference
```

## Versioning

Plugin versions are bumped via the **Bump Plugin Version** GitHub Actions workflow.

Go to **Actions → Bump Plugin Version → Run workflow** and fill in:

| Input | Description |
|-------|-------------|
| `plugin` | Plugin folder name (e.g. `asana-workflow`) |
| `level` | `patch` · `minor` · `major` |

The workflow updates `plugin.json` and the matching entry in `marketplace.json`, tags the commit `<plugin>@<version>`, and pushes directly to the branch it was triggered from.

**Semver guide:**
- `patch` — bug fixes, copy changes, non-breaking skill tweaks (`1.0.0` → `1.0.1`)
- `minor` — new skills or backwards-compatible changes (`1.0.0` → `1.1.0`)
- `major` — breaking changes, removed skills, renamed inputs (`1.0.0` → `2.0.0`)

## License

Proprietary - SIROC Team
