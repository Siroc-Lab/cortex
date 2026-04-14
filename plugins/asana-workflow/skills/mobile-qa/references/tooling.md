# Testing Tool: mobile-mcp

[mobile-mcp](https://github.com/mobile-next/mobile-mcp) — unified API for iOS simulators and Android emulators. Tool names and parameters are self-documented via the MCP schema; this file covers only what the schema doesn't tell you.

## Usage Tips

- **Fetch tool schema first** — Always use `ToolSearch` to load a mobile-mcp tool's schema before calling it for the first time. The schema is small (a few lines) and cheaper than a failed call + retry cycle. Never guess parameter names.
- **Tap before typing** — `mobile_type_keys` requires the input field to be focused first. Tap the field, then type.
- **Coordinates from bounds** — all gestures are coordinate-based. See `investigation.md` → Gestures for coordinate calculation and rules.
- **Android back button** — `mobile_press_button` with `back` is essential for navigation, dismissing dialogs, and closing keyboards. iOS has no equivalent (use UI buttons or swipe gestures).
- **Transient UI** — snackbars, toasts, and alerts disappear quickly. Screenshot immediately when they appear.
- **Evidence persistence** — at assertion points, use `mobile_save_screenshot` (not `mobile_take_screenshot`) to save to the evidence directory. For recordings, pass an explicit `output` path in the evidence directory. See `../../generic-qa/process.md` → Evidence Directory.

## Verification

1. `mobile_list_available_devices` — must return a device list.
2. `mobile_take_screenshot` — must return a screen image.

**Blocking** — cannot proceed without both succeeding. Do NOT attempt to work around this using native shell tools (`xcrun simctl`, `adb`, etc.) — mobile-mcp is a hard dependency, not optional. If it is unavailable, stop and follow the recovery steps below.

### Prerequisites (check only for the target platform)

- **iOS:** `xcode-select -p` and `xcrun simctl list devices available`
- **Android:** `adb version`
- **Node.js:** v22+ (`node --version`)

## MCP Disconnection Recovery

If any mobile-mcp tool fails:

1. Tell the operator.
2. Restart: `npx -y @mobilenext/mobile-mcp@latest &`
3. If still broken, ask operator to run `/mcp` and restart mobile-mcp.
4. Re-verify with `mobile_list_available_devices`.
5. Resume from where you left off.

**On `/resume`:** Always re-verify before continuing.

## Setup Guide

If verification fails:

> mobile-mcp is required but not connected.
>
> **Install:** `npx -y @mobilenext/mobile-mcp@latest`
>
> **Add to MCP settings:**
> ```json
> {"mcpServers":{"mobile-mcp":{"command":"npx","args":["-y","@mobilenext/mobile-mcp@latest"]}}}
> ```

## App State Reset

Reset to clean state before investigation unless operator asks to preserve state.

- **Android:** `adb shell pm clear <package_name>` then `mobile_launch_app`
- **iOS:** No `pm clear` equivalent — uninstall and reinstall:
  ```bash
  xcrun simctl get_app_container booted <bundle_id>  # find .app path
  xcrun simctl uninstall booted <bundle_id>
  xcrun simctl install booted /path/to/App.app
  ```

**Skip reset when:** operator says to keep state, verify mode, or state-dependent investigation.
