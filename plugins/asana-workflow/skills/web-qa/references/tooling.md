# Testing Tool: Chrome DevTools MCP

## Verification

Call `listPages` to verify Chrome DevTools MCP is connected and functional. This must succeed before any investigation begins.

**Expected success:** Returns a list of open browser pages/tabs. At least one page should be accessible.

**Failure means:**
- Chrome DevTools MCP is not configured
- Chrome is not running with remote debugging enabled
- The MCP server is not started

## Setup Guide

**HARD GATE** — do not proceed with investigation if the testing tool is not working. If Chrome DevTools MCP is unavailable, immediately run the diagnostics below **before** doing anything else.

### When Chrome DevTools MCP is unavailable

Run this first:

```bash
which node 2>/dev/null && node --version
```

- **No `node`:** Node.js v22+ is required — tell the operator they need to install it, then restart Claude Code.
- **`node` found:** Chrome DevTools MCP runs via npx, so Node is fine. Check Chrome instead:
  1. Is Chrome running with remote debugging? If not, tell the operator: `open -na "Google Chrome" --args --remote-debugging-port=9222 --user-data-dir=/tmp/chrome-qa`
  2. Reload the plugin: `/plugin reload asana-workflow`

## Safety Rule: Never Kill Chrome

**Never terminate, kill, or close Chrome** (e.g. `pkill`, `killall`, `osascript quit`) without explicit operator approval. The operator may have open work, sessions, or tabs that would be lost. If Chrome needs to be restarted to enable remote debugging, ask first.

## Taking Screenshots

Use Chrome DevTools MCP screenshot capabilities to capture evidence at key moments:
- Before reproducing the issue (baseline state)
- During reproduction (the problematic state)
- After any state changes relevant to the investigation

Use `take_screenshot` with `filePath` to save directly to `$EVIDENCE_DIR`:

```
take_screenshot filePath: "$EVIDENCE_DIR/01-initial-state.png"
```

For small captures (< 2MB) without a filePath it returns base64 — always use `filePath` for evidence that needs to be uploaded. Use descriptive, ordered names (`01-`, `02-`, etc.). Always include screenshots in the report — they are primary evidence.

## Screen Recording

Use `screencast_start` / `screencast_stop` for issues involving:
- Transitions or animations
- Multi-step flows where timing matters
- Flicker, flash, or transient visual bugs
- Race conditions visible in the UI

**Requires:** `ffmpeg` installed and in PATH. If missing, fall back to screenshots only.

**Start recording** (saves to `$EVIDENCE_DIR`):
```
screencast_start path: "$EVIDENCE_DIR/recording.mp4"
```

**Stop recording:**
```
screencast_stop
```

The tool produces an MP4 file directly. Include it in the evidence upload alongside the assertion-point screenshot.
