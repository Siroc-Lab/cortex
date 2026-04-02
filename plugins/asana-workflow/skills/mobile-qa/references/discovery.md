# SUT Discovery

The SUT (System Under Test) is a running mobile application on a simulator, emulator, or physical device. The skill must identify both the app and the target device before investigation begins.

## Discovery Sources

Check these in order:

1. **`$ARGUMENTS`** — If the operator passed an app bundle ID, app name, or device identifier, use it directly. Still confirm it's running.

2. **Project files** — Look for:
   - `.xcodeproj` / `.xcworkspace` — iOS app, check for bundle identifier in project settings
   - `Info.plist` — iOS bundle identifier (`CFBundleIdentifier`)
   - `build.gradle` / `build.gradle.kts` — Android app, check for `applicationId`
   - `AndroidManifest.xml` — Android package name
   - `app.json` / `app.config.js` — React Native / Expo app identifiers

3. **CLAUDE.md** — Look for:
   - App identifiers or bundle IDs
   - Build and run commands (`xcodebuild`, `./gradlew`, `npx expo`)
   - Simulator/emulator instructions
   - Device setup notes

4. **Running devices** — Probe the testing MCP for connected devices, simulators, or emulators. If a list-devices tool is available, call it to see what's connected.

## Confirmation Flow

**Never assume.** After gathering information, present findings:

> "I found `com.example.myapp` in the Xcode project, and there's an iPhone 16 Pro simulator running. Is this the app and device to test, or should I use something different?"

If no SUT information is found:

> "I couldn't determine which app to test or what device to use. Can you provide the app bundle ID and tell me which simulator/emulator/device to target?"

If the operator provides an identifier, verify the app is installed and launchable on the target device.

## SUT is Blocking

Investigation cannot begin without a confirmed, reachable SUT. If the app cannot be launched or the device is not available:
- Report the failure clearly
- Suggest troubleshooting (is the simulator running? is the app installed? is the device connected?)
- Ask the operator for help

Do NOT proceed with source-code-only analysis and call it "investigation."
