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
