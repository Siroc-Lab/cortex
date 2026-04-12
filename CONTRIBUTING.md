# Contributing to SIROC Cortex

## One-time setup

1. Clone the repo
2. Run the setup script — validates GitHub CLI auth, SSH config, and sets `ASANA_PERSONAL_ACCESS_TOKEN` in your shell profile:
   ```bash
   bash setup.sh
   ```
3. In Claude Code, add the local marketplace and install the plugin:
   ```
   /plugin marketplace add /path/to/cortex
   /plugin install asana-workflow@siroc-cortex
   ```

## Development loop

Skills are Markdown files — no build step. After editing a file, start a new Claude Code conversation and changes are picked up automatically. If you're already in a session, run:

```
/plugin reload asana-workflow
```

## Editing an existing skill

Skills live at `plugins/asana-workflow/skills/<name>/SKILL.md`. Edit the file directly.

Keep SKILL.md under ~100 lines. For larger reference content, create a `references/` subdirectory and link to it from SKILL.md.

## Adding a new skill

1. Create `plugins/asana-workflow/skills/<name>/SKILL.md` with YAML frontmatter:
   ```yaml
   ---
   name: skill-name
   description: one-line description of when this skill triggers
   ---
   ```
2. Start a new Claude Code session or run `/plugin reload asana-workflow` to test

## Git workflow

- Never commit directly to `main`
- Create a feature branch, make your changes, open a PR to `main`
- Use conventional commits: `type(scope): description`
  - Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `style`, `perf`

## Versioning

Never edit version numbers in `plugin.json` or `marketplace.json` manually. Version bumps are done via GitHub Actions:

**Actions → Bump Plugin Version → Run workflow**

| Input | Description |
|-------|-------------|
| `plugin` | Plugin folder name (e.g. `asana-workflow`) |
| `level` | `patch` · `minor` · `major` |

- `patch` — bug fixes, copy tweaks, non-breaking skill changes
- `minor` — new skills, backwards-compatible changes
- `major` — breaking changes, removed or renamed skills
