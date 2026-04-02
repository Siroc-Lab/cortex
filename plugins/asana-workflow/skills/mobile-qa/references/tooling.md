# Testing Tool: Mobile Testing MCP

## Capability Contract

A mobile testing MCP must provide these capabilities for the QA skill to function. Tool names vary by MCP server — examples shown are from common servers.

### Required Capabilities

| Capability | Purpose | Example Tool |
|---|---|---|
| Screenshot | Capture visual evidence | `mobile_take_screenshot` |
| List UI elements | Inspect the accessibility tree | `mobile_list_elements_on_screen` |
| Tap | Interact with elements | `mobile_click_on_screen_at_coordinates` |
| Swipe | Scroll and navigate | `mobile_swipe_on_screen` |
| Type text | Fill inputs | `mobile_type_keys` |
| Launch app | Start or restart the SUT | `mobile_launch_app` |
| Terminate app | Reset app state | `mobile_terminate_app` |

### Optional Capabilities

| Capability | Purpose | Example Tool |
|---|---|---|
| List devices | Discover available simulators/emulators/devices | `mobile_list_available_devices` |
| Screen recording | Capture motion/transitions | `mobile_start_recording` |
| Long press | Context menus, drag initiation | `mobile_long_press_on_screen_at_coordinates` |
| Double tap | Zoom, selection | `mobile_double_tap_on_screen` |
| Set orientation | Test landscape/portrait | `mobile_set_orientation` |

## Verification

Attempt to call the screenshot or list-devices tool. If it returns a result, the MCP is working. If it fails, the testing tool is not connected.

**Expected success:** A screenshot image or a list of connected devices/simulators.

**Failure means:**
- No mobile testing MCP is configured
- No simulator/emulator/device is running
- The MCP server is not started

## Setup Guide

If verification fails, tell the operator:

> A mobile testing MCP is required but not connected. You'll need:
>
> 1. A mobile testing MCP server installed and configured in Claude Code
> 2. A running simulator, emulator, or physical device
> 3. The target app installed on the device
>
> Some options (not prescriptive — choose what fits your setup):
> - [mobile-mcp](https://github.com/mobile-next/mobile-mcp) — cross-platform, most popular
> - [Appium MCP](https://github.com/appium/appium-mcp) — richest tool set, hybrid app support
> - [iOS Simulator MCP](https://github.com/joshuayoes/ios-simulator-mcp) — lightweight, simulator-only
> - [XcodeBuildMCP](https://github.com/getsentry/XcodeBuildMCP) — build+test+debug combo
>
> Once configured, I can verify the connection and proceed.

Do NOT proceed with investigation if the testing tool is not working. This is a **blocking** requirement.

## Taking Screenshots

Use the MCP's screenshot capability to capture evidence at key moments:
- Before reproducing the issue (baseline state)
- During reproduction (the problematic state)
- After any state changes relevant to the investigation

Always include screenshots in the report — they are primary evidence. Mobile investigations are especially screenshot-heavy since accessibility tree data alone often lacks the visual context needed to understand issues.
