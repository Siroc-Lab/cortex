# Installing the SIROC Cortex Plugin

## Quick Start

Run the setup script — it validates prerequisites, configures tokens, and guides you through plugin installation:

```bash
bash setup.sh
```

## What the Script Does

### Step 1: GitHub CLI

Checks that the `gh` CLI is installed, that you're authenticated (`gh auth status`), and that your account can access the private `Siroc-Lab/cortex` repo. If anything is missing it tells you exactly what to run.

### Step 2: Git SSH Configuration

Tests SSH authentication to GitHub (`ssh -T git@github.com`). If you use SSH keys, it offers to configure the HTTPS-to-SSH rewrite that Claude Code needs:

```bash
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

### Step 3: Asana Personal Access Token

Looks for `ASANA_PERSONAL_ACCESS_TOKEN` in your environment or shell profile. If it's missing, prompts you to paste one (generated at https://app.asana.com/0/my-apps) and writes it to your profile. Then validates the token against the Asana API.

### Step 4: GitHub Token for Auto-Updates

Checks for `GITHUB_TOKEN` or `GH_TOKEN` — needed so the marketplace can pull updates in the background. If neither is set, it can extract one from `gh auth token` and save it to your profile.

### Step 5: Plugin Installation Instructions

Once all prerequisites pass, the script fetches the marketplace manifest and prints the exact Claude Code commands to run:

1. **Add the marketplace:** `/plugin marketplace add Siroc-Lab/cortex`
2. **Install plugins:** `/plugin install <plugin-name>@siroc-cortex`

**Note:** If the script added tokens to your shell profile, reload your terminal (`source ~/.zshrc` or open a new window) before continuing.

## Available Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/start-task` | Paste an Asana URL | Validates task, creates branch, draft PR, routes to feature-dev or debugging |
| `/ship-it` | "ship it", "we're done" | Orchestrates pre-checks, summary, PR creation, and Asana update |
| `/pre-ship-check` | "am I ready to ship" | Validates git state, lint, build, and tests |
| `/git-check` | "check git status" | Branch safety, working tree, debug artifacts |
| `/work-summary` | "summarize my work" | Session recap for standups, handoffs, PRs |
| `/create-pr` | "create a PR" | Full PR lifecycle with Asana linking |
| `/asana-api` | Any Asana operation | Asana REST API patterns and operations |

## Updating

To pull the latest plugin version:

```
/plugin marketplace update siroc-cortex
/plugin update asana-workflow@siroc-cortex
```
