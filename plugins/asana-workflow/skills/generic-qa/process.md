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
