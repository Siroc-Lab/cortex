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

## License

Proprietary - SIROC Team
