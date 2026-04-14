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
> 1. Start Chrome with remote debugging (isolated instance): `open -na "Google Chrome" --args --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-qa`
> 2. Reload the plugin (`/plugin reload asana-workflow`) to ensure the MCP server started
>
> Once configured, I can verify the connection and proceed.

Do NOT proceed with investigation if the testing tool is not working. This is a **blocking** requirement.

## Safety Rule: Never Kill Chrome

**Never terminate, kill, or close Chrome** (e.g. `pkill`, `killall`, `osascript quit`) without explicit operator approval. The operator may have open work, sessions, or tabs that would be lost. If Chrome needs to be restarted to enable remote debugging, ask first.

## Taking Screenshots

Use Chrome DevTools MCP screenshot capabilities to capture evidence at key moments:
- Before reproducing the issue (baseline state)
- During reproduction (the problematic state)
- After any state changes relevant to the investigation

Chrome DevTools MCP returns screenshots as base64 PNG data. Save each to `$EVIDENCE_DIR`:

```bash
echo "<base64_data>" | base64 -d > "$EVIDENCE_DIR/01-initial-state.png"
```

Use descriptive, ordered names (`01-`, `02-`, etc.). Always include screenshots in the report — they are primary evidence.

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

Save the key assertion-point frame to `$EVIDENCE_DIR` (same base64 decode approach as screenshots). Include the frame in the report when visual motion is relevant to the finding.
