# Generic QA Abstraction — Design Spec

## Problem

The current `project-qa` skill is hardcoded to web testing via Chrome DevTools MCP. We need a platform-agnostic QA process that can be extended to web, iOS, Android, or any future platform by swapping in platform-specific bindings.

## Architecture: Base + Platform Extensions

Three directories under `plugins/asana-workflow/skills/`:

| Directory | Type | Registered in plugin.json | Invocable |
|---|---|---|---|
| `generic-qa/` | Shared markdown files | No | No |
| `web-qa/` | Skill (`0.1.0`) | Yes | Yes |
| `mobile-qa/` | Skill (`0.1.0`) | Yes | Yes (placeholder) |

`project-qa/` is deleted and replaced by `web-qa`. Plugin version bumps `1.0.5` → `1.1.0`.

## generic-qa/ — Shared Library (Not a Skill)

Plain markdown directory. No SKILL.md, no frontmatter, no registration. Extension skills reference these files at invocation time.

### Files

```
generic-qa/
  process.md                  ← universal QA flow
  references/
    reporting.md              ← confidence levels, report structure
    investigation.md          ← generic observe → reproduce → evidence
```

### process.md — The Universal Flow

Two invocation modes, selected by `$ARGUMENTS`:

**Investigate (default):** Operator has a question about a running app. Output is a report with answer, root cause, reproduction steps, and evidence.

**Verify:** Replay previous reproduction steps and check if behavior now matches expected. Output is Pass or Fail with evidence.

**Prerequisites (abstract — extensions provide concrete details):**
- Testing tool — a working MCP that can observe and interact with the SUT. Blocking.
- SUT — a running application, identified by the extension's discovery process. Blocking.
- Source code — optional, enhances findings with file/line context.

**The 6-step flow:**

1. **Discover the SUT** — read the extension's `references/discovery.md` for platform-specific discovery sources. Infer from project context, always confirm with operator.
2. **Verify Testing Tool** — read the extension's `references/tooling.md` for verification steps. Blocking — cannot proceed without a working tool.
3. **Onboard** — scan source code if available, confirm the SUT, the question, and any relevant context with the operator. Hard gate: operator must confirm before investigating.
4. **Investigate / Verify** — use the extension's `references/investigation.md` for platform-specific observation techniques. In investigate mode: reproduce, trace root cause, capture evidence. In verify mode: replay repro steps, check expected vs actual.
5. **Report** — see `references/reporting.md`. Universal structure: answer, root cause (with confidence level), reproduction steps, evidence, source context (if available), recommendation.
6. **Post to Asana** — only in investigate mode when a task GID is in context. Post structured finding (Bug Confirmed or Cannot Reproduce) via `asana-api` skill.

**Behavior rules (universal):**
1. Never guess silently — infer, then confirm with operator.
2. Testing tool is blocking — no investigation without a working tool.
3. SUT is blocking — no investigation without a confirmed running app.
4. Source code is optional — enhances findings but isn't required.
5. Never modify the SUT — read-only observation.
6. Never skip a blocker silently — report it and ask for help.

**Red flags (universal):**
- Reading source code without having opened the app
- Reporting "Likely" without attempting reproduction
- Accepting a tooling failure without telling the operator
- Investigating beyond what the operator asked
- Hitting a blocker and caveating the report instead of asking for help

### references/reporting.md

Confidence levels:

| Level | Definition | Required Evidence |
|---|---|---|
| Confirmed | Reproduced with evidence | Screenshot, recording, log output, or trace showing the issue |
| Likely | Strong reasoning, not fully reproduced | Partial evidence + explanation of what couldn't be verified and why |
| Suspicion | Noticed during investigation, unverified | Description + what would need to happen to verify |

Report structure (every report):
1. Answer — direct answer to the operator's question
2. Root cause — with confidence level, file/line references if source available
3. Reproduction steps — numbered, specific, followable by anyone
4. Evidence — screenshots, recordings, log output, traces
5. Source context (if available) — file/line references
6. Recommendation — suggested fix or next steps

Asana posting format:
- Bug Confirmed: prefix `🔍 Bug Confirmed`, include root cause, repro steps, evidence summary, recommendation
- Cannot Reproduce: prefix `❓ Cannot Reproduce`, include what was tried, what was observed, environment, questions

### references/investigation.md

Generic investigation techniques (no platform-specific tool names):

- **Navigate** — go to the relevant screen/page/view
- **Observe** — check the rendered state for visual anomalies, missing elements, incorrect states
- **Interact** — trigger the flow the operator asked about (tap, click, fill, submit)
- **Check logs** — look for errors, warnings, exceptions in whatever log output the platform provides
- **Inspect elements** — check element states, properties, visibility via whatever inspection the platform provides
- **Capture evidence** — screenshot at key moments (before, during, after the issue); record if the issue involves motion or transitions

Source code cross-reference (when available):
1. Observe the behavior in the SUT first
2. Identify the relevant component from the screen/URL/view
3. Read the source to understand the logic
4. Trace the code path that produces the observed behavior
5. Include file:line references in the report

Completion criteria:
- The question is answered with evidence, or
- The root cause is identified with a confidence level, or
- Blocked — the operator has been told why

## web-qa/ — Web Extension

Replaces the current `project-qa` skill. Contains only web-specific bindings.

### SKILL.md

```yaml
---
name: web-qa
version: 0.1.0
description: >
  Use when the operator asks to investigate a specific question or problem in a running web application —
  triggers include "QA this", "why is X broken", "test this flow", "/qa", "/web-qa", or any specific question
  about a web app's behavior. Requires a running web app (local or remote) and Chrome DevTools MCP.
argument-hint: <url-of-sut>
---
```

Body: "Read and follow `../generic-qa/process.md` as the QA process. This skill provides the web platform bindings below." Then points to its three reference files. (All cross-skill references use `../generic-qa/` relative paths from the extension's skill directory.)

### references/tooling.md

Chrome DevTools MCP verification:
- Call `list_pages` to verify connected
- Setup guide for Chrome with remote debugging
- Screenshot capabilities
- `experimentalScreencast` for transitions/animations
- Blocking — cannot proceed without it

Content: today's `project-qa/references/tooling.md` with the "Future: Swappable Tooling" section removed (that's now the whole architecture).

### references/discovery.md

SUT is a URL. Discovery sources (in order):
1. `$ARGUMENTS` — if operator passed a URL
2. CLAUDE.md — dev server commands, port numbers, staging URLs
3. package.json scripts — `dev`, `start`, `serve`
4. docker-compose.yml — port mappings
5. docs/ — setup guides

Framework inference table (Vite → 5173, Next.js → 3000, etc.).

Always confirm: "I found `npm run dev` which starts on port 3000. Is this the app to test?"

Content: today's `project-qa/references/sut-discovery.md` unchanged.

### references/investigation.md

Web-specific observation techniques:
- Page navigation — navigate to relevant route
- DOM interaction — click buttons, fill forms, trigger flows
- Console monitoring — JS errors, warnings, uncaught exceptions
- Network inspection — API calls, response codes, payloads, timing
- DOM inspection — element states (disabled, hidden, aria), computed styles

Content: today's `project-qa/references/investigation.md` unchanged.

## mobile-qa/ — Mobile Extension (Placeholder)

Placeholder for future contributors. Provides the structure and capability contract without recommending specific MCP servers.

### SKILL.md

```yaml
---
name: mobile-qa
version: 0.1.0
description: >
  Use when the operator asks to investigate a specific question or problem in a running mobile application —
  triggers include "/mobile-qa", "QA this app", "test this on mobile", or any specific question about
  a mobile app's behavior. Requires a running app on a simulator/emulator or device and a mobile testing MCP.
argument-hint: <app-bundle-id-or-device>
---
```

Body: "Read and follow `../generic-qa/process.md` as the QA process. This skill provides the mobile platform bindings below." Then points to its three reference files.

### references/tooling.md

Capability contract — a mobile testing MCP must provide:
- Screenshot (e.g., `mobile_take_screenshot` or equivalent)
- List UI elements (e.g., `mobile_list_elements_on_screen` or equivalent)
- Tap/swipe/type (e.g., `mobile_click_on_screen_at_coordinates`, `mobile_swipe_on_screen`, `mobile_type_keys` or equivalents)
- App lifecycle (e.g., `mobile_launch_app`, `mobile_terminate_app` or equivalents)

Verification: attempt to call the screenshot or list-devices tool. If it succeeds, proceed. If not, tell the operator a mobile testing MCP is needed.

No specific MCP server recommended. Known options in the ecosystem (for contributor reference, not prescriptive):
- mobile-mcp (mobile-next) — cross-platform, most popular
- Appium MCP (official) — richest tool set, hybrid app support
- iOS Simulator MCP (joshuayoes) — lightweight, simulator-only
- XcodeBuildMCP (getsentry) — build+test+debug combo
- Various Android ADB servers

### references/discovery.md

SUT is an app, not a URL. Discovery sources (in order):
1. `$ARGUMENTS` — app bundle ID, app name, or device identifier
2. Project files — `.xcodeproj`, `.xcworkspace`, `build.gradle`, `AndroidManifest.xml`, `Info.plist`
3. CLAUDE.md — app identifiers, build commands, simulator/emulator instructions
4. Running devices — probe the MCP for connected devices/simulators/emulators

Always confirm: "I found `com.example.app` targeting an iPhone 16 simulator. Is this the app to test?"

### references/investigation.md

Mobile-specific observation techniques:
- Accessibility tree inspection (instead of DOM) — element labels, types, states, bounds
- Gesture interaction — tap, swipe, long-press, pinch (richer than web click/type)
- Device logs — platform log output (instead of browser console)
- Screenshot-heavy workflow — mobile MCPs rely more on visual evidence than structured data
- No direct network inspection in most mobile MCPs — note as limitation, suggest proxy tools if needed

## Integration Changes

### start-task

Update Steps 10a–10c: replace `project-qa` invocations with `web-qa`. Future enhancement: start-task could infer which extension to invoke based on project type, but for now it defaults to `web-qa`.

### plugin.json

- Remove `project-qa` from skills list
- Add `web-qa` and `mobile-qa`
- Bump version `1.0.5` → `1.1.0`

### CLAUDE.md (plugin)

Update the skill relationship diagram:
- `project-qa` → `web-qa` in the start-task tree
- Add `generic-qa` as a shared reference (noted as "not a skill")
- Add `mobile-qa` as a new skill

## What's NOT In Scope

- Splitting `mobile-qa` into `ios-qa` and `android-qa` — future work when contributors need it
- Recommending specific MCP servers for mobile — contributors choose their own
- Changes to reporting format or confidence levels — these are universal and unchanged
- Changes to the Asana integration — unchanged, still conditional on task GID
