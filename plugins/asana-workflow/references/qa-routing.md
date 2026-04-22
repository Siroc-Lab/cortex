# QA Routing Reference

Plugin-level shared reference for resolving which QA skill to invoke and for the start-task QA sub-flow (Steps 10a–10e). Used by both `start-task` (at Step 10) and `ship-it` (at Step 2 if a QA advisory fires). Lives at `plugins/asana-workflow/references/qa-routing.md` alongside other shared references (e.g., `board-resolution.md`).

---

## Resolving the QA Skill

Determine which QA skill to invoke. Applies to **all task categories** — bugs use it for investigate/verify, non-bugs use it for completion verification.

Check in order:

1. **CLAUDE.md** — look for a `qa-skill:` declaration (e.g., `qa-skill: web-qa`, `qa-skill: mobile-qa`, or `qa-skill: none`). If found, use it.
2. **Project signals** — infer from project files:
   - `package.json` (without React Native), `vite.config.*`, `next.config.*` → `web-qa`
   - `.xcodeproj`, `.xcworkspace`, `Info.plist` → `mobile-qa`
   - `build.gradle`, `build.gradle.kts`, `AndroidManifest.xml` → `mobile-qa`
   - `app.json` / `app.config.js` with React Native/Expo → `mobile-qa`
   - No UI framework detected (pure backend, CLI, API, library, infrastructure) → `none`
3. **Ambiguous** — ask the operator (blocking):
   > "Which QA skill should I use?
   > 1. `web-qa` (browser-based, Chrome DevTools MCP)
   > 2. `mobile-qa` (simulator/emulator/device, mobile testing MCP)
   > 3. `none` (no visual UI to verify — backend, API, CLI, library)"

Reuse the resolved QA skill for all QA invocations in the task.

**If `none`:** for bug tasks, skip `Verify Bug` and `Verify Fix` (no visual QA to run) but still run `Fix Bug` with ticket-only context. For non-bug tasks, skip the completion-verification step.

---

## start-task Step 10 Sub-flow

Steps 10a–10e execute only when `$ARGUMENTS` does **not** contain `fast`. Fast mode skips the entire sub-flow and implements inline.

### Step 10a: Resolve QA Skill

Run the resolution logic above. Record the result for the rest of the task.

**If the resolved skill is `none` and the task is non-bug:** skip straight to Step 11.

### Step 10b: Verify Bug

**Bug tasks only, when QA skill != `none`.**

Invoke the resolved QA skill in **investigate** mode with:
- Bug description from the Asana ticket (as the question)
- SUT identifier (URL or app bundle ID, if known from CLAUDE.md or task notes)

Outcomes:
- **Confirmed** (bug reproduced with evidence) → QA skill posts the report to the Asana task. Proceed to Step 10c, passing the full report as context.
- **Cannot reproduce** → **stop**. Report to operator. Let them decide: fix SUT setup, clarify the bug description, or skip verification and proceed to debugging anyway.

### Step 10c: Fix Bug

**Bug tasks only.**

Invoke `fix-bug` with the QA report from Step 10b as enriched context (reproduction steps, evidence, root-cause analysis from runtime observation). If Step 10b was skipped (QA skill is `none`), invoke `fix-bug` with just the Asana ticket context.

`fix-bug` returns after root-cause investigation + TDD pass. It does **not** verify or ship — that is start-task's responsibility (Steps 10d and 11).

### Step 10d: Verify Fix (BLOCKING)

**Bug tasks only, when QA skill != `none`. Cannot be skipped.**

After `fix-bug` returns, re-invoke the resolved QA skill in **verify** mode with the original reproduction steps from Step 10b. The QA skill will rebuild, deploy, and replay the steps.

- **Pass** → QA skill posts `✅ QA Verification — PASSED` to Asana with evidence. Proceed to Step 11.
- **Fail** → QA skill posts `❌ QA Verification — FAILED` to Asana with evidence. Return to Step 10c for another debugging pass.

### Step 10e: QA Verification (Non-Bug Tasks)

**Non-bug tasks only.** Bug tasks already have QA via Steps 10b/10d.

**HARD GATE — always stop and wait for the operator's answer. Auto mode's "minimize interruptions" directive does NOT override this step.**

Skip asking only if the operator has already provided an explicit answer about QA in this session — e.g., passed `skip QA` in the start-task arguments, or said "skip QA" / "run QA" earlier in the conversation. Inferred triviality (small change, simple fix, XS sizing) is NOT a valid reason to skip.

After the development workflow signals completion, ask:

> "Implementation is complete. The changes can be visually verified before shipping — I'll build, deploy to the simulator/browser, and check the affected flows. A screenshot or video will be uploaded to the Asana task as proof of completion.
>
> Run QA verification? [yes / skip]"

Wait for the operator's answer before continuing.

If **yes** — resolve the QA skill if not already resolved (Step 10a) and invoke it with a summary of what was built/changed. The QA skill verifies the implementation, then posts `✅ QA Verification — Feature Complete` to Asana with evidence.

If **skip** — proceed to Step 11. ship-it will offer one more chance if no QA evidence is found (see ship-it Step 2).

---

## ship-it Step 2 Use (Reference)

`ship-it` uses the "Resolving the QA Skill" section above at its Step 2 (QA Verification) when pre-ship-check reports a QA advisory. The same logic, the same order — no redefinition.
