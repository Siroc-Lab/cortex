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
| `start-task-steps` | Full lifecycle variant of start-task with mandatory checkpoint tracking for long or interruptible tasks |
| `ship-it` | Orchestrates pre-checks, summary, PR creation, and Asana update |
| `pre-ship-check` | Validates git state, lint, build, and tests |
| `git-check` | Branch safety, working tree cleanliness, debug artifact detection |
| `work-summary` | Session recap for standups, handoffs, and PRs |
| `create-pr` | Full PR lifecycle with Asana linking and reviewer assignment |
| `asana-api` | Asana REST API patterns and common operations |
| `log-task` | Creates an Asana task from work discovered or completed in conversation |
| `fix-bug` | Full bug-fix lifecycle orchestrator: root cause investigation, TDD hard gate, and ship |
| `mobile-qa` | Investigates and verifies bugs in iOS simulators and Android emulators via mobile-mcp |
| `web-qa` | Investigates and verifies bugs in running web applications via Chrome DevTools MCP |

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
    .mcp.json             # MCP server config (wires up mobile-mcp)
    skills/
      start-task/         # Entry point skill + references
      start-task-steps/   # Checkpoint-tracked lifecycle variant
      ship-it/            # Shipping orchestrator
      pre-ship-check/     # Readiness gate
      git-check/          # Git state validation
      work-summary/       # Session recap
      create-pr/          # PR lifecycle
      asana-api/          # Asana API operations + spec reference
      fix-bug/            # Bug-fix lifecycle orchestrator
      generic-qa/         # Shared QA process and references (used by web-qa, mobile-qa)
      web-qa/             # Web QA via Chrome DevTools MCP
      mobile-qa/          # Mobile QA via mobile-mcp (iOS simulators + Android emulators)
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

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for setup instructions and the development workflow.

## License

Proprietary - SIROC Team
