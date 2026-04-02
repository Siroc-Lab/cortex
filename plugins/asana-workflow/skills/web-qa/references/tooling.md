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
