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

## How It Works

The typical development lifecycle looks like this:

```
/start-task <asana-url>   →   develop (feature-dev / fix-bug)   →   /ship-it
```

`start-task` is the main entry point — it validates the task, sets up the branch and draft PR, and routes to the right workflow automatically. When development is done, it hands off to `ship-it`, which runs checks, creates the PR, and closes the Asana ticket. All other skills can also be used standalone.

## Available Skills

### Workflow entry points

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/start-task` | Paste an Asana URL, "start task", "work on this" | Validates task, creates branch and draft PR, routes to feature-dev or fix-bug. Options: `fast` (implement inline, skip sub-skill routing), `brainstorm` (brainstorm design first), `feature-dev` (go straight to implementation). Supports pause ("park this", "I'm blocked") and resume. |
| `/start-task-steps` | Same triggers as start-task | Checkpoint-tracked variant for long or complex tasks — writes a checkpoint after every step so work can be safely interrupted and resumed. |
| `/log-task` | "log this as a task", "create a ticket", "capture this in Asana" | Creates an Asana task from work discovered or completed mid-conversation. Arguments: `sprint: <url>`, `backlog: <url>` to override board defaults. |

### Shipping

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/ship-it` | "ship it", "we're done", "ready to ship" | Orchestrates pre-ship checks, work summary, PR creation/promotion, and Asana task close. |
| `/create-pr` | "create a PR", "open a pull request" | Full PR lifecycle with Asana linking and reviewer assignment. |

### Validation & checks

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/pre-ship-check` | "am I ready to ship", "pre-flight check" | Validates git state, lint, build, and tests before shipping. |
| `/git-check` | "check git status", "is my branch clean" | Branch safety, working tree cleanliness, debug artifact detection. |

### QA

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/web-qa` | "QA this", "why is X broken", `/web-qa <url>` | Investigates and verifies bugs in running web applications via Chrome DevTools MCP. |
| `/mobile-qa` | "/mobile-qa", "QA this app" | Investigates and verifies bugs in iOS simulators and Android emulators via mobile-mcp. |

### Utilities

| Skill | Trigger | Description |
|-------|---------|-------------|
| `/work-summary` | "summarize my work", "standup notes", "session recap" | Generates a concise recap of what was done, for standups, handoffs, or PRs. |
| `/fix-bug` | Invoked automatically by start-task for Bug tickets | Full bug-fix lifecycle: root cause investigation, TDD hard gate, and ship. |
| `/asana-api` | Any Asana operation | Asana REST API patterns and common operations. |

## Updating

To pull the latest plugin version:

```
/plugin marketplace update siroc-cortex
/plugin update asana-workflow@siroc-cortex
```
