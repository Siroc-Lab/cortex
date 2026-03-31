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
