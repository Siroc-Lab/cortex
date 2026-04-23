┌──────────────────────────────────────────────────────────────────────────┐
│                 start-task  ·  Complete Flow Reference                   │
│          /start-task <url>  [brainstorm | feature-dev | fast]            │
└──────────────────────────────────────────────────────────────────────────┘

 LEGEND
   ┌──────┐  action / step          ◆  decision point
   └──────┘                         ▼  flow direction
   ■  terminal state                ↺  loop back
   [A]  jump connector             ··  skips steps (shortcut)

 CHECKPOINTING (always on, every run)
   File:  ~/.claude/asana-workflow/checkpoints/<task-gid>.md
   Writes go through plugins/asana-workflow/skills/start-task/scripts/checkpoint.sh
   Every numbered step is recorded. See references/checkpoints.md for details.


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
   5. checkpoint.sh block <gid> <step> <reason>
      checkpoint.sh append-note <gid> <blocking-question + who to follow up with>
   6. git push WIP branch
   7. confirm: commit hash · Asana comment link · checkpoint path
   task stays "In Progress"  →  ■ PAUSED
══════════════════════════════════════════════════════════════════════════════


                              MAIN FLOW

   NOTE: $ARGUMENTS is parsed once up front into named flags:
         fast_mode  · workflow_choice (brainstorm|feature-dev)
         Steps 0–10 and Step 12 run in every mode. Only Step 11 changes
         (skipped when fast_mode; every QA row → State=skipped).

                                  │
                                  ▼
             ┌────────────────────────────────────────────┐
             │  INIT / RESUME  (before Step 0)            │
             │  ls ~/.claude/asana-workflow/checkpoints/  │
             │     <task-gid>.md                          │
             └───────────────────┬────────────────────────┘
                                 │
                        ◆ checkpoint exists?
                       ╱                     ╲
                     YES                      NO
                      │                        │
                RESUME FLOW              checkpoint.sh init <gid> <url>
                      │                        │
              ┌───────┴────────┐               │
              │ find first     │               │
              │ non-terminal   │               │
              │ row (Completed │               │
              │ != [x] AND     │               │
              │ State !=       │               │
              │ skipped)       │               │
              │                │               │
              │ verify branch, │               │
              │ check task     │               │
              │ status, fetch  │               │
              │ new Asana      │               │
              │ comments since │               │
              │ last_updated   │               │
              │ if row=blocked │               │
              │                │               │
              │ post resume    │               │
              │ comment if     │               │
              │ previous state │               │
              │ was blocked    │               │
              └───────┬────────┘               │
                      │                        │
             ·· [A] SKIP to first            │
                non-terminal step ··         │
                      │                        │
                      └──────────────┬─────────┘
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
             │  checkpoint.sh set <gid> task_id <task-id> │
             └───────────────────┬────────────────────────┘
                                 │
                                 ▼
             ┌────────────────────────────────────────────┐
             │  3 · Validate Sprint-Readiness             │
             │                                            │
             │  ◻ Active sprint membership  BLOCKING      │
             │    (must be in current sprint in Asana)    │
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
                          │  6a · Worktree? (BLOCKING)                 │
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
                          │  6b · Base Branch? (BLOCKING)              │
                          │  main (default) or specify other           │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  7 · Create Feature Branch                 │
                          │  <task-id>/<slug>  off base branch         │
                          │  inform only — no question                 │
                          │  checkpoint.sh set <gid> branch / base     │
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
                          │  9a · Move to In Progress                  │
                          │  Asana section move → "In Progress"        │
                          │  if move fails: report, do not block       │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                          ┌────────────────────────────────────────────┐
                          │  9b · Post Start Comment                   │
                          │  post 🏁 start comment (deduplicate)       │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼

══════════════════════════════════════════════════════════════════════════════
 STEP 10 · ROUTE TO WORKFLOW
══════════════════════════════════════════════════════════════════════════════

                                         ◆ fast_mode?
                                        ╱             ╲
                                      YES              NO
                                       │                │
                                       │                │
                          ┌────────────┴──────────┐     │
                          │  implement inline      │    │
                          │  Read · Edit · Bash    │    │
                          │  no skill invoked      │    │
                          │                        │    │
                          │  Step 11 (QA):         │    │
                          │  checkpoint.sh skip    │    │
                          │  for every QA row      │    │
                          │  with reason           │    │
                          │  "fast mode"           │    │
                          └────────────┬──────────┘    │
                                       │                ▼
                                       │           ◆ Category?
                                       │         ╱     │     ╲
                                       │       Bug  Non-bug  missing
                                       │        │     │        │
                                       │        │     │    ask: bug or
                                       │        │     │    non-bug?
                                       │        │     │        │
                                       │        │     └────┬───┘
                                       │        │          │
                                       │        │    ◆ workflow_choice?
                                       │        │   ╱     │       ╲
                                       │        │  brn   fdev     none
                                       │        │   │     │         │
                                       │        │  sup:  feat-dev  ask
                                       │        │  brn   :feature  brn
                                       │        │   │     -dev     /
                                       │        │   │     │        fdev
                                       │        │   │     │         │
                                       │        │   └─────┴────┬────┘
                                       │        │              │
                                       │        │       (external skill,
                                       │        │        returns to Step 11)
                                       │        │              │
                                       │        └── bug-path   │
                                       │            to Step 11 │
                                       │               (QA)    │
                                       │                │      │
                                       │                ▼      ▼
                                       │        ┌──── Step 11 ────┐
                                       │        │  QA SUB-FLOW    │
                                       │        │  (see below)    │
                                       │        └────────┬────────┘
                                       │                 │
                                       │                 │
                                       └─────────┬───────┘
                                                 │
                                                 ▼
                          ┌────────────────────────────────────────────┐
                          │  12 · Ship It                              │
                          │  triggers: QA sub-flow completes / skipped │
                          │  or fast mode direct to ship               │
                          │  do not wait for user to ask               │
                          │                                            │
                          │  pre-ship-check (owns QA gate)             │
                          │    · Step 1d: QA verification              │
                          │      (prompts operator if no QA evidence   │
                          │       for non-bug tasks)                   │
                          │    · git state + lint + build + tests      │
                          │  work-summary                              │
                          │  promote draft PR → ready                  │
                          │  move task → "In Review"                   │
                          │  post completion comment                   │
                          │                                            │
                          │  on success:                               │
                          │  checkpoint.sh delete <gid>                │
                          └───────────────────┬────────────────────────┘
                                              │
                                              ▼
                                           ■ DONE


══════════════════════════════════════════════════════════════════════════════
 STEP 11 · QA SUB-FLOW  ·  entered from Step 10 routing
══════════════════════════════════════════════════════════════════════════════

  Details in plugins/asana-workflow/references/qa-routing.md
  Skipped entirely when fast_mode (every row → State=skipped).

                          ┌────────────────────────────────────────────┐
                          │  QA: Resolve                               │
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
                          │     no UI framework         → none         │
                          │  3. ambiguous → ask: web-qa / mobile-qa /  │
                          │                      none                  │
                          └───────────────────┬────────────────────────┘
                                              │
                                  ◆ resolved skill == none
                                    AND non-bug task?
                                 ╱                       ╲
                               YES                       NO
                                │                         │
                          skip to Step 12                 │
                          (nothing to verify)             │
                                                          ▼
                                                    ◆ Category?
                                                  ╱              ╲
                                                Bug            Non-bug
                                                 │                │
                                                 ▼                │
                          ┌────────────────────────────────────┐ │
                          │  QA: Investigate Bug               │ │
                          │  (skip if skill == none)           │ │
                          │  QA skill: investigate mode        │ │
                          │  input: bug description + SUT      │ │
                          └───────────────┬────────────────────┘ │
                                          │                      │
                                    ◆ reproduced?                │
                                   ╱               ╲             │
                             CONFIRMED          CANNOT REPRODUCE │
                                  │                   │          │
                         post QA report to       operator decides│
                         Asana task               · fix SUT      │
                                  │               · clarify      │
                                  │                    → ■ STOP  │
                                  │               · skip verify, │
                                  │                 proceed      │
                                  │                              │
                                  ▼                              │
                          ┌────────────────────────────────────┐ │
                          │  QA: Fix Bug                       │ │
                          │  invoke fix-bug skill              │ │
                          │  pass QA report as context         │ │
                          └───────────────┬────────────────────┘ │
                                          │                      │
                                          ▼                      │
                          ┌────────────────────────────────────┐ │
                          │  QA: Verify Fix  (BLOCKING)        │ │
                          │  (skip if skill == none)           │ │
                          │  QA skill: verify mode             │ │
                          │  input: repro steps from earlier   │ │
                          └───────────────┬────────────────────┘ │
                                          │                      │
                                     ◆ fixed?                    │
                                    ╱         ╲                  │
                                  PASS         FAIL              │
                                   │             │               │
                                   │        ↺ loop back          │
                                   │        to QA: Fix Bug       │
                                   │                             │
                                   └────────────┬────────────────┘
                                                │
                                     (to Step 12 · Ship It)
                                                │
                                                ▼
                          Non-bug path:
                          ┌────────────────────────────────────┐
                          │  QA: Verify Non-Bug  (HARD GATE)   │
                          │  asked after development completes │
                          │                                    │
                          │  "Implementation complete.         │
                          │   Run QA verification? [yes/skip]" │
                          └───────────────┬────────────────────┘
                                          │
                                     ◆ operator?
                                    ╱              ╲
                                  YES              SKIP
                                   │                 │
                          QA skill: verify       proceed to
                          mode                   Step 12 (ship-it
                          posts ✅ QA            pre-ship-check
                          Feature Complete       will offer one
                          to Asana               more chance)
                                   │                 │
                                   └────────┬────────┘
                                            │
                                 (to Step 12 · Ship It)
