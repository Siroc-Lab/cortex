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

> **Placeholder:** This skill defines the structure and capability contract but is not yet wired into `start-task` routing. Contributors should customize `references/tooling.md` with their preferred MCP server. `start-task` currently defaults to `web-qa` for bug verification.

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
