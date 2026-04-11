┌──────────────────────────────────────────────────────────────────────────┐
│                 start-task  ·  Complete Flow Reference                   │
│          /start-task <url>  [brainstorm | feature-dev | fast]            │
└──────────────────────────────────────────────────────────────────────────┘

 LEGEND
   ┌──────┐  action / step          ◆  decision point
   └──────┘                         ▼  flow direction
   ■  terminal state                ↺  loop back
   [A]  jump connector             ··  skips steps (shortcut)


══════════════════════════════════════════════════════════════════════════════
 PAUSE FLOW  ·  fires at any point during the session

 Trigger phrases: "park this" · "I'm blocked" · "pause task" ·
 "put this on hold" · "waiting on X" · "blocked by X" · "save my progress" ·
 "need to wait for an answer" · "come back to this later" ·
 "pick this up later"

   1. verify current branch matches task (warn if merge in progress)
   2. git add -A && git commit  →  WIP: <task-id> — blocked on [reason]
   3. draft blocking question  →  USER MUST APPROVE wording before posting
   4. post to Asana via asana-api (with @mention of blocker)
   5. write .claude/checkpoints/<task-id>.md  (gitignored, local only)
   6. git push WIP branch
   7. confirm: commit hash · Asana comment link · checkpoint path
   task stays "In Progress"  →  ■ PAUSED
══════════════════════════════════════════════════════════════════════════════


                              MAIN FLOW

   NOTE: if $ARGUMENTS contains "fast", note it at the start.
   Steps 0–9 and Step 11 run normally. Only Step 10 changes (inline impl).

                                  │
                                  ▼
             ┌────────────────────────────────────────────┐
             │  0 · Skill Dependencies  (advisory)        │
             │  feature-dev@claude-plugins-official       │
             │  superpowers@claude-plugins-official       │
             └───────────────────┬────────────────────────┘
                                 │
                            ◆ installed?
                           ╱             ╲
                         NO               YES
                          │                │
                    warn + ask             │
                    ╱        ╲            │
                install    continue        │
                    └──────────►──────────┘
                                 │
                                 ▼
             ┌────────────────────────────────────────────┐
             │  1 · Get Task URL                          │
             │  parse $ARGUMENTS → extract task GID       │
             └───────────────────┬────────────────────────┘
                                 │
                            ◆ valid URL?
                           ╱           ╲
                         NO             YES
                          │              │
                        prompt           │
                          └─────────────┘
                                 │
                                 ▼
             ┌────────────────────────────────────────────┐
             │  2 · Fetch Task Details                    │
             │  name · category · sprint · assignee       │
             │  custom fields · notes · memberships       │
             │  → print summary                           │
             └───────────────────┬────────────────────────┘
                                 │
                                 ▼
             ┌────────────────────────────────────────────┐
             │  3 · Validate Sprint-Readiness             │
             │                                            │
             │  ◻ Sprint membership  BLOCKING             │
             │    (must be added in Asana manually)       │
             │  ◻ Estimated time     BLOCKING  → API fix  │
             │  ◻ Product Status     BLOCKING  → API fix  │
             │  ◻ ID field           WARN (skippable)     │
             └───────────────────┬────────────────────────┘
                                 │
                            ◆ all blocking pass?
                           ╱                   ╲
                         NO                     YES
                          │                      │
                   report checklist              │
                   offer API fix                 │
                          ↺────────────────────┘
                                 │
                                 ▼
             ┌────────────────────────────────────────────┐
             │  4 · Fetch Subtasks                        │
             │  group: incomplete (remaining) / done      │
             └───────────────────┬────────────────────────┘
                                 │
                                 ▼
             ┌────────────────────────────────────────────┐
             │  5 · Fetch Comments & Attachments          │
             │  filter stories → comments                 │
             │  list attachments · note images/mockups    │
             └───────────────────┬────────────────────────┘
                                 │
                                 ▼
             ┌────────────────────────────────────────────┐
             │  6 · Check for Existing Work               │
             │  git fetch --prune                         │
             │  check branches + open PRs by task ID      │
             └───────────────────┬────────────────────────┘
                                 │
                        ◆ branch / PR exists?
                       ╱                     ╲
                     YES                      NO
                      │                        │
             offer: resume / fresh             │
                 ╱          ╲                 │
            RESUME          FRESH             │
                │              └──────────────┤
         checkout                             │
         existing branch                      │
                │                             │
                ▼                             ▼
             ┌────────────────────────────────────────────┐
             │  6a · Check for Checkpoint                 │
             │  .claude/checkpoints/<task-id>.md          │
             └───────────────────┬────────────────────────┘
                                 │
                        ◆ checkpoint found?
                       ╱                     ╲
                     YES                      NO
                      │                        │
          ┌───────────┴──────────────────┐     │
          │                              │     │
          │  ① verify branch exists      │     │
          │    ◆ branch deleted?          │     │
          │      YES → offer:            │     │
          │        · recreate from base  │     │
          │        · start fresh ─────────────►┤ (del checkpoint, → 6b)
          │      NO → continue           │     │
          │                              │     │
          │  ② check task status         │     │
          │    warn if completed,        │     │
          │    reassigned, or moved      │     │
          │    since paused_at           │     │
          │                              │     │
          │  ③ fetch new Asana comments  │     │
          │    since paused_at           │     │
          │    ◆ no new comments?        │     │
          │      YES → offer:            │     │
          │        · resume anyway       │     │
          │        · keep waiting        │     │
          │          → ■ STOP            │     │
          │      NO → present comments   │     │
          │                              │     │
          └───────────┬──────────────────┘     │
                      │                        │
               ◆ resume?                       │
              ╱                  ╲            │
            YES              NO (start fresh)  │
             │                    └────────────┤
          checkout                            │
          branch (if not already)             │
          post resume comment                 │
          delete checkpoint file              │
             │                               │
             ·· [A] SKIP to Step 10 ··       │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  6b · Worktree? (BLOCKING)                 │
                          │  worktree (isolated) or current dir        │
                          └───────────────────┬────────────────────────┘
                                              │
                                         ◆ worktree?
                                        ╱           ╲
                                      YES            NO
                                       │              │
                                 EnterWorktree        │
                                       └─────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  6c · Base Branch? (BLOCKING)              │
                          │  main (default) or specify other           │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  7 · Create Feature Branch                 │
                          │  <task-id>/<slug>  off base branch         │
                          │  inform only — no question                 │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  8 · Create Draft PR                       │
                          │  empty commit → push branch                │
                          │  gh pr create --draft                      │
                          │  capture PR URL (threaded to ship-it)      │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  9 · Move to In Progress + Comment         │
                          │  Asana section move → "In Progress"        │
                          │  if move fails: report, do not block       │
                          │  post 🏁 start comment (deduplicate)       │
                          └───────────────────┬────────────────────────┘
                                              │
                                   [A] resume re-enters here
                             (on resume: workflow is predetermined
                              from checkpoint  workflow  field —
                              skip the routing decision tree below)
                                              │
                                              ▼

══════════════════════════════════════════════════════════════════════════════
 STEP 10 · ROUTE TO WORKFLOW
══════════════════════════════════════════════════════════════════════════════

                                         ◆ $ARGS contains fast?
                                        ╱                       ╲
                                      YES                        NO
                                       │                          │
                          ┌────────────┴──────────┐         ◆ Category?
                          │  implement inline      │        ╱    │      ╲
                          │  Read · Edit · Bash    │      Bug  Feature  missing
                          │  no skill invoked      │       │      │       │
                          └────────────┬──────────┘       │      │    ask: bug
                                       │                  │      │    or feature?
                                       │                  │      │       │
                                       │                  │      └───┬───┘
                                       │                  │          │
                                       │                  │     ◆ $ARGS workflow?
                                       │                  │    ╱     │      ╲
                                       │                  │  brn    fdev   (none)
                                       │                  │   │      │       │
                                       │                  │  sup:  feat-   ask →
                                       │                  │  brn   dev:fd brn/fdev
                                       │                  │   │      │       │
                                       │                  │   └──────┴───────┘
                                       │                  │   (external skills,
                                       │                  │    return to Step 11)
                                       │                  │
                                       │                  └──── Bug Flow (below) ──►
                                       │
                                       ▼
                          ┌────────────────────────────────────────────┐
                          │  11 · Ship It                              │
                          │  triggers: feature-dev Phase 7 (Summary)  │
                          │  or systematic-debugging post-verify       │
                          │  do not wait for user to ask               │
                          │                                            │
                          │  pre-ship-check                            │
                          │  work-summary                              │
                          │  promote draft PR → ready                  │
                          │  move task → "In Review"                   │
                          │  post completion comment                   │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                                           ■ DONE


══════════════════════════════════════════════════════════════════════════════
 BUG FLOW  ·  entered from Step 10 when Category = Bug  →  returns to Step 11
══════════════════════════════════════════════════════════════════════════════

                          ┌────────────────────────────────────────────┐
                          │  10a · Resolve QA Skill                    │
                          │                                            │
                          │  1. CLAUDE.md  qa-skill: declaration       │
                          │  2. project signals:                       │
                          │     package.json (no React Native) /       │
                          │     vite.config / next.config  → web-qa   │
                          │     .xcodeproj / .xcworkspace /            │
                          │     Info.plist / build.gradle /            │
                          │     build.gradle.kts /                     │
                          │     AndroidManifest.xml      → mobile-qa   │
                          │     app.json / app.config.js               │
                          │     + React Native / Expo    → mobile-qa   │
                          │  3. ambiguous → ask: web-qa or mobile-qa   │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  10b · Verify Bug                          │
                          │  QA skill: investigate mode                │
                          │  input: bug description + SUT identifier   │
                          └───────────────────┬────────────────────────┘
                                              │
                                        ◆ reproduced?
                                       ╱               ╲
                                 CONFIRMED          CANNOT REPRODUCE
                                      │                   │
                             post QA report to       operator decides:
                             Asana task               · fix SUT setup
                                      │               · clarify ticket
                                      │                    → ■ STOP
                                      │               · skip verification
                                      │                    → proceed to 10c
                                      │                      (no repro evidence)
                                      ▼
                          ┌────────────────────────────────────────────┐
                          │  10c · Fix Bug                             │
                          │  invoke fix-bug skill                      │
                          │  pass QA report as enriched context        │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  10d · Verify Fix                          │
                          │  QA skill: verify mode                     │
                          │  input: original repro steps from 10b      │
                          └───────────────────┬────────────────────────┘
                                              │
                                         ◆ fixed?
                                        ╱         ╲
                                      PASS         FAIL
                                       │             │
                                  → Step 11     ↺ loop back
                                   (above)       to Step 10c
