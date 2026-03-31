# Generic QA Abstraction Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Refactor the monolithic `project-qa` skill into a base+extension architecture: `generic-qa/` (shared markdown), `web-qa/` (web extension), `mobile-qa/` (placeholder mobile extension).

**Architecture:** `generic-qa/` is a plain directory of markdown files (not a skill) defining the universal QA process. `web-qa/` and `mobile-qa/` are registered skills whose SKILL.md files reference `../generic-qa/process.md` for the shared flow, then provide platform-specific tooling, discovery, and investigation references.

**Tech Stack:** Claude Code skills (markdown), asana-workflow plugin

---

## File Structure

```
plugins/asana-workflow/skills/
  generic-qa/                          ← NEW: shared markdown library (not a skill)
    process.md                         ← universal QA flow extracted from project-qa/SKILL.md
    references/
      reporting.md                     ← universal report structure (from project-qa)
      investigation.md                 ← generic investigation techniques (new, platform-agnostic)
  web-qa/                              ← NEW: web extension skill (replaces project-qa)
    SKILL.md                           ← skill frontmatter + pointer to generic-qa/process.md
    references/
      tooling.md                       ← Chrome DevTools MCP (from project-qa, minus "Future" section)
      discovery.md                     ← URL-based SUT discovery (from project-qa/sut-discovery.md)
      investigation.md                 ← web-specific techniques (from project-qa/investigation.md)
  mobile-qa/                           ← NEW: mobile extension skill (placeholder)
    SKILL.md                           ← skill frontmatter + pointer to generic-qa/process.md
    references/
      tooling.md                       ← capability contract, no specific MCP
      discovery.md                     ← app-based SUT discovery
      investigation.md                 ← mobile-specific techniques
  project-qa/                          ← DELETED (replaced by generic-qa + web-qa)
```

**Modified files:**
- `plugins/asana-workflow/.claude-plugin/plugin.json` — version bump, skill list update
- `plugins/asana-workflow/CLAUDE.md` — update structure diagram and skill relationships
- `plugins/asana-workflow/skills/start-task/SKILL.md` — `project-qa` → `web-qa` references
- `plugins/asana-workflow/skills/start-task/references/skill-dependencies.md` — `project-qa` → `web-qa` in bundled skills table

---

### Task 1: Create generic-qa/process.md

Extract the universal QA flow from `project-qa/SKILL.md`, removing all web-specific content.

**Files:**
- Create: `plugins/asana-workflow/skills/generic-qa/process.md`
- Reference: `plugins/asana-workflow/skills/project-qa/SKILL.md`

- [ ] **Step 1: Create process.md**

```markdown
# QA Process

Universal QA process for investigating questions and problems in running applications. Platform extensions provide the concrete tooling, discovery, and observation bindings.

## Input / Output

Two invocation modes, selected by `$ARGUMENTS`:

### Investigate (default)

**Trigger:** `$ARGUMENTS` is a target (URL, app ID), a question, or empty. Anything that is not `verify:`.

**Input:** A specific question or problem about a running application's behavior.

**Output:** A report with answer, root cause, reproduction steps, and evidence.

### Verify

**Trigger:** `$ARGUMENTS` starts with `verify:` followed by the reproduction steps to replay.

**Input:** A previous report's reproduction steps + the expected behavior to check for.

**Output:** Pass (behavior now matches expected, with evidence) or Fail (behavior still matches the original "actual", with evidence).

## Prerequisites

- **Testing tool** — A working MCP that can observe and interact with the SUT. See the platform extension's `references/tooling.md` for verification steps. **Blocking** — cannot proceed without it.
- **SUT** — A running application, identified by the platform extension's discovery process. See the extension's `references/discovery.md`. **Blocking** — cannot proceed without it.
- **Source code** (optional) — enhances findings with file/line context but is not required.

## The Flow

### Step 1: Discover the SUT

Read the platform extension's `references/discovery.md` for platform-specific discovery sources. Infer from project context (arguments, project files, configuration), then **always confirm with the operator**.

**Never assume.** Present what you found and confirm:

> "I found [SUT details]. Is this what I should test, or should I use something different?"

If the operator's request has no specific question, ask:

> "What specifically should I look at? Give me a page, flow, or behavior to investigate."

### Step 2: Verify Testing Tool

Read the platform extension's `references/tooling.md` for the specific verification steps. Confirm the testing MCP is connected and functional.

**BLOCKING** — cannot proceed without a working testing tool.

### Step 3: Onboard

If source code is available, scan the architecture (entry points, routes, components) to build context for the investigation. This is advisory — it informs but does not replace runtime observation.

Ask the operator about any additional context for the problem.

**HARD GATE:** Confirm with the operator before investigating — the SUT, the question, and any relevant context must be agreed on. If everything looks clear, confirm your understanding.

### Step 4: Investigate

**Investigate mode (default):**

Reproduce the reported behavior by observing the SUT. Use the platform extension's `references/investigation.md` for available observation techniques. Trace the root cause. See `references/investigation.md` (in this directory) for generic investigation guidance.

If source code is available, cross-reference to explain *why* the behavior occurs.

Done when:
- The question is answered with evidence, or
- The root cause is identified with a confidence level, or
- Blocked — the operator has been told why

**Verify mode:**

Replay the reproduction steps from the previous report, step by step. At the assertion point (expected vs actual), capture evidence of the current behavior.

- If behavior now matches **expected** → **Pass**. Include evidence showing the corrected behavior.
- If behavior still matches **actual** → **Fail**. Include evidence showing the issue persists. The fix didn't work.

### Step 5: Report

See `references/reporting.md` (in this directory) for the full report structure. Every report includes:

1. **Answer** — Direct answer to the operator's question
2. **Root cause** — With confidence level (Confirmed / Likely / Suspicion)
3. **Reproduction steps** — Numbered, specific, followable by anyone
4. **Evidence** — Screenshots, recordings, log output, traces
5. **Source context** (if available) — File/line references
6. **Recommendation** — Suggested fix or next steps

### Step 6: Post QA Finding to Asana

Only posts in **investigate** mode when invoked from `start-task` (a task GID is available in context). Verify mode does not post — ship-it's `🤖 Done` already signals the fix landed, and failures loop back to fix-bug with no ticket update needed.

Post a comment via the `asana-api` skill based on the investigation outcome:

#### Bug Confirmed

Prefix: `🔍 Bug Confirmed`

Include:
1. **Root cause** — what's causing the behavior, with confidence level
2. **Reproduction steps** — numbered, specific, followable by anyone
3. **Evidence summary** — what was captured (screenshots, log output, traces)
4. **Recommendation** — suggested fix or next steps

#### Cannot Reproduce

Prefix: `❓ Cannot Reproduce`

Include:
1. **What was tried** — the steps attempted to reproduce the reported behavior
2. **What was observed instead** — actual behavior with evidence
3. **Environment** — SUT identifier, testing tool, any relevant config
4. **Questions** — specific clarifications needed to retry

Format both as structured HTML (Asana rich text). This creates a permanent record on the ticket of what QA found before any fix work begins.

If no task GID is in context (standalone invocation), skip this step — the report is the artifact and the operator decides what to do with it.

## Behavior Rules

1. **Never guess silently** — infer, then confirm with operator.
2. **Testing tool is blocking** — no investigation without a working tool.
3. **SUT is blocking** — no investigation without a confirmed running app.
4. **Source code is optional** — enhances findings but isn't required.
5. **Never modify the SUT** — read-only observation.
6. **Never skip a blocker silently** — report it and ask for help.

## Red Flags

You are off-track if:
- You're reading source code without having opened the app
- You're reporting "Likely" without attempting reproduction
- You accepted a tooling failure without telling the operator
- You're investigating beyond what the operator asked
- You hit a blocker and caveated your report instead of asking for help

## Integration

When invoked by `start-task` for bug tickets, the extension skill participates in a verify → fix → verify loop:

1. **start-task** invokes the QA extension in **investigate** mode with the bug description
2. If bug is **Confirmed**, start-task passes the report to `systematic-debugging`
3. After the fix, start-task re-invokes the QA extension in **verify** mode with the original reproduction steps
4. **Pass** → proceed to ship-it. **Fail** → back to debugging.

If bug **cannot be reproduced**, start-task stops and asks the operator how to proceed.
```

Write this to `plugins/asana-workflow/skills/generic-qa/process.md`.

- [ ] **Step 2: Commit**

```bash
git add plugins/asana-workflow/skills/generic-qa/process.md
git commit -m "feat(generic-qa): extract universal QA process from project-qa"
```

---

### Task 2: Create generic-qa/references/reporting.md

Extract the universal reporting reference from `project-qa/references/reporting.md`.

**Files:**
- Create: `plugins/asana-workflow/skills/generic-qa/references/reporting.md`
- Reference: `plugins/asana-workflow/skills/project-qa/references/reporting.md`

- [ ] **Step 1: Create reporting.md**

Copy `plugins/asana-workflow/skills/project-qa/references/reporting.md` as-is — it's already platform-agnostic. No changes needed.

```markdown
# Reporting

## Confidence Levels

Use exactly one of these for each finding:

| Level | Definition | Required Evidence |
|---|---|---|
| **Confirmed** | Reproduced with evidence | Screenshot, recording, log output, or trace showing the issue |
| **Likely** | Strong reasoning, not fully reproduced | Partial evidence + explanation of what couldn't be verified and why |
| **Suspicion** | Noticed during investigation, unverified | Description of what was observed + what would need to happen to verify |

## Report Structure

Every report follows this structure:

### 1. Answer
Direct answer to the operator's question. Lead with this — no preamble.

### 2. Root Cause (if applicable)
What's causing the behavior. Include confidence level. If source code is available, include file:line references.

### 3. Reproduction Steps
Numbered steps anyone can follow:

```
1. Open the SUT
2. Navigate to [screen/page/view]
3. Interact with [element]
4. **Expected:** [what should happen]
5. **Actual:** [what happens instead]
```

Be specific: exact targets, exact element descriptions, exact inputs. Someone with no context should be able to follow these steps.

### 4. Evidence
Include all captured evidence:

- **Screenshots** — at key moments (before, during, after the issue)
- **Recordings** — for transitions, animations, multi-step flows
- **Log output** — exact error messages with stack traces
- **Traces** — failed requests, unexpected responses, timing issues

### 5. Source Context (if source code available)
File and line references explaining the code path that produces the behavior.

### 6. Recommendation
Suggested fix or next steps. Be specific — name the file, the function, the change.

## Output

The report IS the artifact. The skill takes a question as input and produces the report as output. No side effects — no memory persistence, no ticket creation, no file writes. The operator decides what to do with the report.
```

Write this to `plugins/asana-workflow/skills/generic-qa/references/reporting.md`.

- [ ] **Step 2: Commit**

```bash
git add plugins/asana-workflow/skills/generic-qa/references/reporting.md
git commit -m "feat(generic-qa): add universal reporting reference"
```

---

### Task 3: Create generic-qa/references/investigation.md

Write a new platform-agnostic investigation techniques reference.

**Files:**
- Create: `plugins/asana-workflow/skills/generic-qa/references/investigation.md`

- [ ] **Step 1: Create investigation.md**

```markdown
# Investigation Techniques

## Observation via Testing Tool

All findings come from **observing the running application**. The platform extension's `references/investigation.md` details the specific tools available. These are the universal techniques:

### Navigation
- Go to the relevant screen, page, or view in the SUT
- Observe the rendered state
- Check for visual anomalies, missing elements, incorrect states

### Interaction
- Trigger the flow the operator asked about
- Observe state changes after interaction
- Test the specific behavior in question

### Log Inspection
- Check for errors, warnings, or exceptions in the platform's log output
- Note uncaught errors or unexpected entries
- Correlate log entries with observed behavior

### Element Inspection
- Inspect element states (enabled/disabled, visible/hidden, properties)
- Check element presence or absence
- Verify element attributes relevant to the issue

### Evidence Capture
- Screenshot at key moments: before, during, and after the issue
- Record if the issue involves motion, transitions, or timing
- Capture log output and traces that support findings

## Source Code Cross-Reference

When source code is available in the working directory, use it to **explain** findings — not to replace runtime observation.

**Good:** "The button is disabled because [state] is true. In `src/path/file:45`, the handler sets this state but the error path on line 62 never resets it."

**Bad:** "Looking at the code, the button might be disabled when [state] is true." (No runtime verification.)

### How to Cross-Reference

1. Observe the behavior in the SUT first
2. Identify the relevant component from the screen/view
3. Read the source to understand the logic
4. Trace the specific code path that produces the observed behavior
5. Include file:line references in the report

## Completion Criteria

Investigation is done when:

- **The question is answered** — with evidence
- **Root cause is identified** — with a confidence level and supporting evidence
- **Blocked** — a tooling or access issue prevents further investigation; the operator has been told why

## Anti-Patterns

| Doing this... | Means you're off track |
|---|---|
| Reading source code without opening the app | Investigate the SUT, not the codebase |
| Reporting "likely" without trying to reproduce | Attempt reproduction first |
| Guessing the answer from code patterns | Observe the actual behavior |
| Investigating beyond the operator's question | Stay focused on what was asked |
| Silently giving up on reproduction | Tell the operator and ask for help |
```

Write this to `plugins/asana-workflow/skills/generic-qa/references/investigation.md`.

- [ ] **Step 2: Commit**

```bash
git add plugins/asana-workflow/skills/generic-qa/references/investigation.md
git commit -m "feat(generic-qa): add generic investigation techniques reference"
```

---

### Task 4: Create web-qa/ skill

Create the web extension skill that replaces `project-qa`.

**Files:**
- Create: `plugins/asana-workflow/skills/web-qa/SKILL.md`
- Create: `plugins/asana-workflow/skills/web-qa/references/tooling.md`
- Create: `plugins/asana-workflow/skills/web-qa/references/discovery.md`
- Create: `plugins/asana-workflow/skills/web-qa/references/investigation.md`
- Reference: `plugins/asana-workflow/skills/project-qa/SKILL.md`
- Reference: `plugins/asana-workflow/skills/project-qa/references/tooling.md`
- Reference: `plugins/asana-workflow/skills/project-qa/references/sut-discovery.md`
- Reference: `plugins/asana-workflow/skills/project-qa/references/investigation.md`

- [ ] **Step 1: Create web-qa/SKILL.md**

```markdown
---
name: web-qa
version: 0.1.0
description: >
  Use when the operator asks to investigate a specific question or problem in a running web application —
  triggers include "QA this", "why is X broken", "test this flow", "/qa", "/web-qa", or any specific question
  about a web app's behavior. Requires a running web app (local or remote) and Chrome DevTools MCP.
argument-hint: <url-of-sut>
---

# Web QA

Investigate questions and problems in running **web** applications using Chrome DevTools MCP.

## Base Process

Read and follow `../generic-qa/process.md` as the QA process. This skill provides the web platform bindings below.

## Platform Bindings

- **Testing tool:** Chrome DevTools MCP — see `references/tooling.md`
- **SUT discovery:** URL-based — see `references/discovery.md`
- **Investigation techniques:** DOM, console, network — see `references/investigation.md`

## Reference Files

- **`../generic-qa/process.md`** — Universal QA flow (the process to follow)
- **`../generic-qa/references/reporting.md`** — Confidence levels, report structure
- **`../generic-qa/references/investigation.md`** — Generic investigation guidance
- **`references/tooling.md`** — Chrome DevTools MCP verification, screenshots, screencasts
- **`references/discovery.md`** — URL-based SUT discovery
- **`references/investigation.md`** — Web-specific observation techniques (DOM, console, network)
```

Write this to `plugins/asana-workflow/skills/web-qa/SKILL.md`.

- [ ] **Step 2: Create web-qa/references/tooling.md**

Copy `plugins/asana-workflow/skills/project-qa/references/tooling.md` and remove the "Future: Swappable Tooling" section (that's now the architecture itself).

```markdown
# Testing Tool: Chrome DevTools MCP

## Verification

Call `list_pages` to verify Chrome DevTools MCP is connected and functional. This must succeed before any investigation begins.

**Expected success:** Returns a list of open browser pages/tabs. At least one page should be accessible.

**Failure means:**
- Chrome DevTools MCP is not configured
- Chrome is not running with remote debugging enabled
- The MCP server is not started

## Setup Guide

If verification fails, tell the operator:

> Chrome DevTools MCP is required but not connected. To set it up:
>
> 1. Install the Chrome DevTools MCP server
> 2. Start Chrome with remote debugging: `open -a "Google Chrome" --args --remote-debugging-port=9222`
> 3. Configure the MCP server in your Claude Code settings
>
> Once configured, I can verify the connection and proceed.

Do NOT proceed with investigation if the testing tool is not working. This is a **blocking** requirement.

## Taking Screenshots

Use Chrome DevTools MCP screenshot capabilities to capture evidence at key moments:
- Before reproducing the issue (baseline state)
- During reproduction (the problematic state)
- After any state changes relevant to the investigation

Always include screenshots in the report — they are primary evidence.

## Screen Recording with experimentalScreencast

Use `experimentalScreencast` for issues involving:
- Transitions or animations
- Multi-step flows where timing matters
- Flicker, flash, or transient visual bugs
- Race conditions visible in the UI

**Start recording:**
```
Page.startScreencast with format: "png", quality: 80, everyNthFrame: 2
```

**Stop recording:**
```
Page.stopScreencast
```

Capture frames during the reproduction and assemble into a sequence. Include the recording in the report when visual motion is relevant to the finding.
```

Write this to `plugins/asana-workflow/skills/web-qa/references/tooling.md`.

- [ ] **Step 3: Create web-qa/references/discovery.md**

Copy `plugins/asana-workflow/skills/project-qa/references/sut-discovery.md` unchanged (rename file only).

```markdown
# SUT Discovery

The SUT (System Under Test) is a running web application — either a local dev server or a remote staging/production URL. The skill must identify it before investigation begins.

## Discovery Sources

Check these in order:

1. **`$ARGUMENTS`** — If the operator passed a URL, use it directly. Still confirm it's reachable.

2. **CLAUDE.md** — Look for:
   - Dev server commands (`npm run dev`, `yarn dev`, `pnpm dev`)
   - Port numbers (`localhost:3000`, `localhost:5173`)
   - Staging/preview URLs
   - Environment setup instructions

3. **package.json scripts** — Look for:
   - `dev`, `start`, `serve`, `start:local` scripts
   - Port configurations in the script commands

4. **docker-compose.yml** — Look for:
   - Service port mappings
   - Frontend service definitions

5. **docs/** — Look for:
   - Setup guides mentioning URLs
   - Architecture docs with service maps

## Inference Patterns

Common patterns to recognize:

| Framework | Default Port | Start Command |
|-----------|-------------|---------------|
| Vite | 5173 | `vite dev` |
| Next.js | 3000 | `next dev` |
| Create React App | 3000 | `react-scripts start` |
| Strapi | 1337 | `strapi develop` |
| Express | 3000 | `node server.js` |

## Confirmation Flow

**Never assume.** After gathering information, present findings:

> "I found `npm run dev` in package.json which starts a Vite server on port 5173. Is this the app to test, or should I use a different URL?"

If no SUT information is found:

> "I couldn't determine where the app is running. Can you provide the URL or tell me how to start the dev server?"

If the operator provides a URL, verify it's reachable using the testing tool (`navigate` to the URL and confirm a page loads).

## SUT is Blocking

Investigation cannot begin without a confirmed, reachable SUT. If the SUT cannot be reached:
- Report the failure clearly
- Suggest troubleshooting (is the server running? correct port? firewall?)
- Ask the operator for help

Do NOT proceed with source-code-only analysis and call it "investigation."
```

Write this to `plugins/asana-workflow/skills/web-qa/references/discovery.md`.

- [ ] **Step 4: Create web-qa/references/investigation.md**

Copy `plugins/asana-workflow/skills/project-qa/references/investigation.md` unchanged.

```markdown
# Investigation Techniques

## Observation via Testing Tool

All findings come from **observing the running application**. The testing tool (Chrome DevTools MCP) provides:

### Page Navigation
- Navigate to the relevant page/route
- Observe the rendered state
- Check for visual anomalies, missing elements, incorrect states

### Interaction
- Click buttons, fill forms, trigger flows
- Observe state changes after interaction
- Test the specific behavior the operator asked about

### Console Monitoring
- Check for JavaScript errors
- Watch for warnings that indicate problems
- Note any uncaught exceptions or promise rejections

### Network Inspection
- Monitor API calls during the flow
- Check response codes, payloads, timing
- Identify failed requests, slow responses, or missing calls

### DOM Inspection
- Inspect element states (disabled, hidden, aria attributes)
- Check computed styles
- Verify element presence/absence in the DOM

## Source Code Cross-Reference

When source code is available in the working directory, use it to **explain** findings — not to replace runtime observation.

**Good:** "The button is disabled because `isSubmitting` state is true. In `src/components/Form.tsx:45`, the submit handler sets this state but the error path on line 62 never resets it."

**Bad:** "Looking at the code, the button might be disabled when `isSubmitting` is true." (No runtime verification.)

### How to Cross-Reference

1. Observe the behavior in the SUT first
2. Identify the relevant component/page from the URL or DOM
3. Read the source to understand the logic
4. Trace the specific code path that produces the observed behavior
5. Include file:line references in the report

## Completion Criteria

Investigation is done when:

- **The question is answered** — with evidence (screenshots, console output, network traces)
- **Root cause is identified** — with a confidence level and supporting evidence
- **Blocked** — a tooling or access issue prevents further investigation; the operator has been told why

## Anti-Patterns

| Doing this... | Means you're off track |
|---|---|
| Reading source code without opening the app | Investigate the SUT, not the codebase |
| Reporting "likely" without trying to reproduce | Attempt reproduction first |
| Guessing the answer from code patterns | Observe the actual behavior |
| Investigating beyond the operator's question | Stay focused on what was asked |
| Silently giving up on reproduction | Tell the operator and ask for help |
```

Write this to `plugins/asana-workflow/skills/web-qa/references/investigation.md`.

- [ ] **Step 5: Commit**

```bash
git add plugins/asana-workflow/skills/web-qa/
git commit -m "feat(web-qa): create web extension skill replacing project-qa"
```

---

### Task 5: Create mobile-qa/ skill (placeholder)

Create the mobile extension skill with the capability contract.

**Files:**
- Create: `plugins/asana-workflow/skills/mobile-qa/SKILL.md`
- Create: `plugins/asana-workflow/skills/mobile-qa/references/tooling.md`
- Create: `plugins/asana-workflow/skills/mobile-qa/references/discovery.md`
- Create: `plugins/asana-workflow/skills/mobile-qa/references/investigation.md`

- [ ] **Step 1: Create mobile-qa/SKILL.md**

```markdown
---
name: mobile-qa
version: 0.1.0
description: >
  Use when the operator asks to investigate a specific question or problem in a running mobile application —
  triggers include "/mobile-qa", "QA this app", "test this on mobile", or any specific question about
  a mobile app's behavior. Requires a running app on a simulator/emulator or device and a mobile testing MCP.
argument-hint: <app-bundle-id-or-device>
---

# Mobile QA

Investigate questions and problems in running **mobile** applications (iOS or Android) using a mobile testing MCP.

## Base Process

Read and follow `../generic-qa/process.md` as the QA process. This skill provides the mobile platform bindings below.

## Platform Bindings

- **Testing tool:** Any mobile testing MCP that satisfies the capability contract — see `references/tooling.md`
- **SUT discovery:** App-based — see `references/discovery.md`
- **Investigation techniques:** Accessibility tree, gestures, device logs — see `references/investigation.md`

## Reference Files

- **`../generic-qa/process.md`** — Universal QA flow (the process to follow)
- **`../generic-qa/references/reporting.md`** — Confidence levels, report structure
- **`../generic-qa/references/investigation.md`** — Generic investigation guidance
- **`references/tooling.md`** — Mobile testing MCP capability contract
- **`references/discovery.md`** — App-based SUT discovery
- **`references/investigation.md`** — Mobile-specific observation techniques
```

Write this to `plugins/asana-workflow/skills/mobile-qa/SKILL.md`.

- [ ] **Step 2: Create mobile-qa/references/tooling.md**

```markdown
# Testing Tool: Mobile Testing MCP

## Capability Contract

A mobile testing MCP must provide these capabilities for the QA skill to function. Tool names vary by MCP server — examples shown are from common servers.

### Required Capabilities

| Capability | Purpose | Example Tool |
|---|---|---|
| Screenshot | Capture visual evidence | `mobile_take_screenshot` |
| List UI elements | Inspect the accessibility tree | `mobile_list_elements_on_screen` |
| Tap | Interact with elements | `mobile_click_on_screen_at_coordinates` |
| Swipe | Scroll and navigate | `mobile_swipe_on_screen` |
| Type text | Fill inputs | `mobile_type_keys` |
| Launch app | Start or restart the SUT | `mobile_launch_app` |
| Terminate app | Reset app state | `mobile_terminate_app` |

### Optional Capabilities

| Capability | Purpose | Example Tool |
|---|---|---|
| List devices | Discover available simulators/emulators/devices | `mobile_list_available_devices` |
| Screen recording | Capture motion/transitions | `mobile_start_recording` |
| Long press | Context menus, drag initiation | `mobile_long_press_on_screen_at_coordinates` |
| Double tap | Zoom, selection | `mobile_double_tap_on_screen` |
| Set orientation | Test landscape/portrait | `mobile_set_orientation` |

## Verification

Attempt to call the screenshot or list-devices tool. If it returns a result, the MCP is working. If it fails, the testing tool is not connected.

**Expected success:** A screenshot image or a list of connected devices/simulators.

**Failure means:**
- No mobile testing MCP is configured
- No simulator/emulator/device is running
- The MCP server is not started

## Setup Guide

If verification fails, tell the operator:

> A mobile testing MCP is required but not connected. You'll need:
>
> 1. A mobile testing MCP server installed and configured in Claude Code
> 2. A running simulator, emulator, or physical device
> 3. The target app installed on the device
>
> Some options (not prescriptive — choose what fits your setup):
> - [mobile-mcp](https://github.com/mobile-next/mobile-mcp) — cross-platform, most popular
> - [Appium MCP](https://github.com/appium/appium-mcp) — richest tool set, hybrid app support
> - [iOS Simulator MCP](https://github.com/joshuayoes/ios-simulator-mcp) — lightweight, simulator-only
> - [XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP) — build+test+debug combo
>
> Once configured, I can verify the connection and proceed.

Do NOT proceed with investigation if the testing tool is not working. This is a **blocking** requirement.

## Taking Screenshots

Use the MCP's screenshot capability to capture evidence at key moments:
- Before reproducing the issue (baseline state)
- During reproduction (the problematic state)
- After any state changes relevant to the investigation

Always include screenshots in the report — they are primary evidence. Mobile investigations are especially screenshot-heavy since accessibility tree data alone often lacks the visual context needed to understand issues.
```

Write this to `plugins/asana-workflow/skills/mobile-qa/references/tooling.md`.

- [ ] **Step 3: Create mobile-qa/references/discovery.md**

```markdown
# SUT Discovery

The SUT (System Under Test) is a running mobile application on a simulator, emulator, or physical device. The skill must identify both the app and the target device before investigation begins.

## Discovery Sources

Check these in order:

1. **`$ARGUMENTS`** — If the operator passed an app bundle ID, app name, or device identifier, use it directly. Still confirm it's running.

2. **Project files** — Look for:
   - `.xcodeproj` / `.xcworkspace` — iOS app, check for bundle identifier in project settings
   - `Info.plist` — iOS bundle identifier (`CFBundleIdentifier`)
   - `build.gradle` / `build.gradle.kts` — Android app, check for `applicationId`
   - `AndroidManifest.xml` — Android package name
   - `app.json` / `app.config.js` — React Native / Expo app identifiers

3. **CLAUDE.md** — Look for:
   - App identifiers or bundle IDs
   - Build and run commands (`xcodebuild`, `./gradlew`, `npx expo`)
   - Simulator/emulator instructions
   - Device setup notes

4. **Running devices** — Probe the testing MCP for connected devices, simulators, or emulators. If a list-devices tool is available, call it to see what's connected.

## Confirmation Flow

**Never assume.** After gathering information, present findings:

> "I found `com.example.myapp` in the Xcode project, and there's an iPhone 16 Pro simulator running. Is this the app and device to test, or should I use something different?"

If no SUT information is found:

> "I couldn't determine which app to test or what device to use. Can you provide the app bundle ID and tell me which simulator/emulator/device to target?"

If the operator provides an identifier, verify the app is installed and launchable on the target device.

## SUT is Blocking

Investigation cannot begin without a confirmed, reachable SUT. If the app cannot be launched or the device is not available:
- Report the failure clearly
- Suggest troubleshooting (is the simulator running? is the app installed? is the device connected?)
- Ask the operator for help

Do NOT proceed with source-code-only analysis and call it "investigation."
```

Write this to `plugins/asana-workflow/skills/mobile-qa/references/discovery.md`.

- [ ] **Step 4: Create mobile-qa/references/investigation.md**

```markdown
# Investigation Techniques

## Observation via Testing Tool

All findings come from **observing the running application**. The mobile testing MCP provides platform-specific observation capabilities:

### Accessibility Tree Inspection
- List all visible UI elements with their labels, types, states, and bounds
- This is the mobile equivalent of DOM inspection — the primary way to understand what's on screen
- Check element states: enabled/disabled, selected, focused, accessibility labels
- Use element bounds (coordinates) to inform tap/swipe actions

### Gesture Interaction
- **Tap** — press buttons, select items, navigate
- **Swipe** — scroll lists, navigate between screens, dismiss modals
- **Long press** — trigger context menus, initiate drag operations
- **Pinch** — zoom in/out (if supported by MCP)
- **Type** — fill text inputs, search fields

Gestures are coordinate-based in most mobile MCPs. Use the accessibility tree to find element bounds, then tap at the center of the element's bounding box.

### Device Logs
- Check platform log output for errors, warnings, crashes
- Mobile equivalent of browser console monitoring
- Look for unhandled exceptions, assertion failures, crash reports
- Correlate log entries with the interaction that triggered them

### Screenshot-Heavy Workflow
Mobile MCPs provide less structured data than web DevTools. Compensate with frequent screenshots:
- Before and after each interaction
- When something looks wrong
- To capture transient states (loading, animations, transitions)

Screenshots are the primary evidence format for mobile investigations.

### Known Limitations
- **No direct network inspection** — most mobile MCPs cannot intercept network traffic. If network behavior is relevant to the investigation, note this limitation and suggest the operator use a proxy tool (e.g., Charles Proxy, mitmproxy) for network capture.
- **No runtime JavaScript/native debugging** — mobile MCPs observe the UI layer, not the runtime. Source code cross-referencing is the primary tool for understanding "why."

## Source Code Cross-Reference

When source code is available in the working directory, use it to **explain** findings — not to replace runtime observation.

### How to Cross-Reference

1. Observe the behavior in the SUT first
2. Identify the relevant screen/view from the accessibility tree or navigation state
3. Read the source to understand the logic (view controllers, activities, components)
4. Trace the specific code path that produces the observed behavior
5. Include file:line references in the report

## Completion Criteria

Investigation is done when:

- **The question is answered** — with evidence (screenshots, accessibility tree data, device logs)
- **Root cause is identified** — with a confidence level and supporting evidence
- **Blocked** — a tooling or access issue prevents further investigation; the operator has been told why

## Anti-Patterns

| Doing this... | Means you're off track |
|---|---|
| Reading source code without opening the app | Investigate the SUT, not the codebase |
| Reporting "likely" without trying to reproduce | Attempt reproduction first |
| Guessing the answer from code patterns | Observe the actual behavior |
| Tapping at hardcoded coordinates without checking the accessibility tree | Elements move — always get fresh bounds |
| Investigating beyond the operator's question | Stay focused on what was asked |
| Silently giving up on reproduction | Tell the operator and ask for help |
```

Write this to `plugins/asana-workflow/skills/mobile-qa/references/investigation.md`.

- [ ] **Step 5: Commit**

```bash
git add plugins/asana-workflow/skills/mobile-qa/
git commit -m "feat(mobile-qa): create mobile extension skill (placeholder)"
```

---

### Task 6: Delete project-qa/ and update integrations

Remove the old skill and update all references.

**Files:**
- Delete: `plugins/asana-workflow/skills/project-qa/` (entire directory)
- Modify: `plugins/asana-workflow/.claude-plugin/plugin.json`
- Modify: `plugins/asana-workflow/CLAUDE.md`
- Modify: `plugins/asana-workflow/skills/start-task/SKILL.md:26` — prerequisites list
- Modify: `plugins/asana-workflow/skills/start-task/SKILL.md:154-167` — Steps 10a-10c
- Modify: `plugins/asana-workflow/skills/start-task/references/skill-dependencies.md:78` — bundled skills table

- [ ] **Step 1: Delete project-qa directory**

```bash
rm -rf plugins/asana-workflow/skills/project-qa/
```

- [ ] **Step 2: Update plugin.json**

Read the current file, then update. The current content is:

```json
{
  "name": "asana-workflow",
  "description": "End-to-end Asana-driven development workflow: from ticket to shipped PR with automated task tracking, git management, and team communication",
  "version": "1.0.5",
  "author": {
    "name": "SIROC Team"
  }
}
```

Replace with:

```json
{
  "name": "asana-workflow",
  "description": "End-to-end Asana-driven development workflow: from ticket to shipped PR with automated task tracking, git management, and team communication",
  "version": "1.1.0",
  "author": {
    "name": "SIROC Team"
  }
}
```

- [ ] **Step 3: Update CLAUDE.md plugin structure**

In `plugins/asana-workflow/CLAUDE.md`, replace the structure section:

Old:
```
    ├── project-qa/        ← QA investigation & verification (bundled)
```

New:
```
    ├── generic-qa/        ← Shared QA process & references (not a skill — used by web-qa, mobile-qa)
    ├── web-qa/            ← Web QA investigation & verification (bundled)
    ├── mobile-qa/         ← Mobile QA investigation & verification (bundled, placeholder)
```

- [ ] **Step 4: Update CLAUDE.md skill relationships**

In `plugins/asana-workflow/CLAUDE.md`, replace in the relationship diagram:

Old:
```
  ├── project-qa         (verify bug → verify fix loop)
```

New:
```
  ├── web-qa             (verify bug → verify fix loop, extends generic-qa)
```

And add after the `ship-it` tree:

```
generic-qa (shared markdown, not a skill)
  ├── process.md         (universal QA flow)
  └── references/        (reporting, investigation)

web-qa (extends generic-qa)
  └── references/        (Chrome DevTools MCP tooling, URL discovery, DOM/console/network)

mobile-qa (extends generic-qa, placeholder)
  └── references/        (mobile MCP capability contract, app discovery, accessibility tree/gestures)
```

- [ ] **Step 5: Update start-task/SKILL.md prerequisites**

In `plugins/asana-workflow/skills/start-task/SKILL.md` line 26, replace:

Old:
```
- Access to `feature-dev:feature-dev`, `superpowers:systematic-debugging`, `project-qa`, and optionally `superpowers:brainstorming` skills
```

New:
```
- Access to `feature-dev:feature-dev`, `superpowers:systematic-debugging`, `web-qa`, and optionally `superpowers:brainstorming` skills
```

- [ ] **Step 6: Update start-task/SKILL.md Steps 10a-10c**

In `plugins/asana-workflow/skills/start-task/SKILL.md`, replace Steps 10a-10c:

Old (lines 154-167):
```markdown
### Step 10a: Verify Bug (Bug category only)

Invoke `project-qa` in **investigate** mode with the bug description from the Asana ticket as the question and the SUT URL (if known from CLAUDE.md or task notes).

- **Confirmed** (bug reproduced with evidence) → project-qa posts the QA report to the Asana task (Step 6 in project-qa). Proceed to Step 10b, passing the full report as context.
- **Cannot reproduce** → **stop**. Tell the operator the bug could not be reproduced. Let them decide: fix SUT setup, clarify the bug description, or skip verification and proceed to debugging anyway.

### Step 10b: Fix Bug

Invoke `fix-bug` with the project-qa report as enriched context. This gives the debugger richer context than the ticket alone — reproduction steps, evidence, and root cause analysis from runtime observation.

### Step 10c: Verify Fix

After the fix is committed, re-invoke `project-qa` in **verify** mode with the original reproduction steps from Step 10a.

- **Pass** (behavior now matches expected) → confirmed fixed, proceed to Step 11.
- **Fail** (behavior still matches original actual) → tell the operator the fix didn't resolve the issue. Return to Step 10b for another debugging pass.
```

New:
```markdown
### Step 10a: Verify Bug (Bug category only)

Invoke `web-qa` in **investigate** mode with the bug description from the Asana ticket as the question and the SUT URL (if known from CLAUDE.md or task notes).

- **Confirmed** (bug reproduced with evidence) → web-qa posts the QA report to the Asana task (Step 6 in the generic-qa process). Proceed to Step 10b, passing the full report as context.
- **Cannot reproduce** → **stop**. Tell the operator the bug could not be reproduced. Let them decide: fix SUT setup, clarify the bug description, or skip verification and proceed to debugging anyway.

### Step 10b: Fix Bug

Invoke `fix-bug` with the web-qa report as enriched context. This gives the debugger richer context than the ticket alone — reproduction steps, evidence, and root cause analysis from runtime observation.

### Step 10c: Verify Fix

After the fix is committed, re-invoke `web-qa` in **verify** mode with the original reproduction steps from Step 10a.

- **Pass** (behavior now matches expected) → confirmed fixed, proceed to Step 11.
- **Fail** (behavior still matches original actual) → tell the operator the fix didn't resolve the issue. Return to Step 10b for another debugging pass.
```

- [ ] **Step 7: Update skill-dependencies.md bundled skills table**

In `plugins/asana-workflow/skills/start-task/references/skill-dependencies.md`, replace in the bundled skills table:

Old:
```
| `asana-workflow:project-qa` | QA investigation & verification |
```

New:
```
| `asana-workflow:web-qa` | Web QA investigation & verification (extends generic-qa) |
| `asana-workflow:mobile-qa` | Mobile QA investigation & verification (extends generic-qa, placeholder) |
```

- [ ] **Step 8: Commit**

```bash
git add -A plugins/asana-workflow/
git commit -m "refactor: replace project-qa with generic-qa + web-qa + mobile-qa

BREAKING: project-qa skill removed, replaced by:
- generic-qa/ (shared process, not a skill)
- web-qa (web extension, same triggers as project-qa)
- mobile-qa (mobile extension, placeholder)

start-task and skill-dependencies updated to reference web-qa."
```

---

### Task 7: Verify and clean up

Final verification that everything is consistent.

**Files:**
- All files created/modified in Tasks 1-6

- [ ] **Step 1: Verify directory structure**

```bash
find plugins/asana-workflow/skills/generic-qa plugins/asana-workflow/skills/web-qa plugins/asana-workflow/skills/mobile-qa -type f | sort
```

Expected:
```
plugins/asana-workflow/skills/generic-qa/process.md
plugins/asana-workflow/skills/generic-qa/references/investigation.md
plugins/asana-workflow/skills/generic-qa/references/reporting.md
plugins/asana-workflow/skills/mobile-qa/SKILL.md
plugins/asana-workflow/skills/mobile-qa/references/discovery.md
plugins/asana-workflow/skills/mobile-qa/references/investigation.md
plugins/asana-workflow/skills/mobile-qa/references/tooling.md
plugins/asana-workflow/skills/web-qa/SKILL.md
plugins/asana-workflow/skills/web-qa/references/discovery.md
plugins/asana-workflow/skills/web-qa/references/investigation.md
plugins/asana-workflow/skills/web-qa/references/tooling.md
```

- [ ] **Step 2: Verify project-qa is fully removed**

```bash
# Should return no results
find plugins/asana-workflow/skills/project-qa -type f 2>/dev/null && echo "ERROR: project-qa still exists" || echo "OK: project-qa removed"
```

- [ ] **Step 3: Grep for stale project-qa references**

```bash
grep -r "project-qa" plugins/asana-workflow/ --include="*.md" --include="*.json"
```

Expected: No results. If any remain, fix them — they should reference `web-qa` or `generic-qa` instead.

- [ ] **Step 4: Verify cross-references resolve**

Check that `../generic-qa/process.md` referenced from `web-qa/SKILL.md` and `mobile-qa/SKILL.md` actually exists:

```bash
ls plugins/asana-workflow/skills/generic-qa/process.md
ls plugins/asana-workflow/skills/generic-qa/references/reporting.md
ls plugins/asana-workflow/skills/generic-qa/references/investigation.md
```

All three should exist.
