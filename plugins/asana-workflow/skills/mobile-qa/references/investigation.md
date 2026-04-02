# Mobile Investigation Techniques

Mobile testing MCP observation capabilities. For generic investigation guidance (source code cross-referencing, completion criteria, anti-patterns), see `../../generic-qa/references/investigation.md`.

## Accessibility Tree Inspection
- List all visible UI elements with their labels, types, states, and bounds
- This is the mobile equivalent of DOM inspection — the primary way to understand what's on screen
- Check element states: enabled/disabled, selected, focused, accessibility labels
- Use element bounds (coordinates) to inform tap/swipe actions

## Gesture Interaction
- **Tap** — press buttons, select items, navigate
- **Swipe** — scroll lists, navigate between screens, dismiss modals
- **Long press** — trigger context menus, initiate drag operations
- **Pinch** — zoom in/out (if supported by MCP)
- **Type** — fill text inputs, search fields

Gestures are coordinate-based in most mobile MCPs. Use the accessibility tree to find element bounds, then tap at the center of the element's bounding box.

## Device Logs
- Check platform log output for errors, warnings, crashes
- Mobile equivalent of browser console monitoring
- Look for unhandled exceptions, assertion failures, crash reports
- Correlate log entries with the interaction that triggered them

## Screenshot-Heavy Workflow
Mobile MCPs provide less structured data than web DevTools. Compensate with frequent screenshots:
- Before and after each interaction
- When something looks wrong
- To capture transient states (loading, animations, transitions)

Screenshots are the primary evidence format for mobile investigations.

## Known Limitations
- **No direct network inspection** — most mobile MCPs cannot intercept network traffic. If network behavior is relevant to the investigation, note this limitation and suggest the operator use a proxy tool (e.g., Charles Proxy, mitmproxy) for network capture.
- **No runtime JavaScript/native debugging** — mobile MCPs observe the UI layer, not the runtime. Source code cross-referencing is the primary tool for understanding "why."

## Mobile-Specific Anti-Pattern

| Doing this... | Means you're off track |
|---|---|
| Tapping at hardcoded coordinates without checking the accessibility tree | Elements move — always get fresh bounds |
