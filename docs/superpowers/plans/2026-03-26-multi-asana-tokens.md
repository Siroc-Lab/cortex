# Multi-Asana-Token Support Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Support multiple named Asana tokens so Claude can switch accounts conversationally while defaulting to `$ASANA_PERSONAL_ACCESS_TOKEN`.

**Architecture:** Token resolution logic lives exclusively in `asana-api/SKILL.md` — the single chokepoint for all Asana API calls. Additional tokens are stored as `ASANA_TOKEN_<NAME>` env vars. `setup.sh` gains an optional loop to register additional accounts.

**Tech Stack:** Bash (setup.sh), Markdown skill files (no runtime code)

---

### Task 1: Create feature branch

**Files:**
- No file changes — git only

- [ ] **Step 1: Create and check out the feature branch**

```bash
git checkout -b feature/multi-asana-tokens
```

Expected output: `Switched to a new branch 'feature/multi-asana-tokens'`

---

### Task 2: Update `asana-api/SKILL.md` — add Token Resolution section

**Files:**
- Modify: `plugins/asana-workflow/skills/asana-api/SKILL.md`

The current file opens with a one-liner ("All operations use bearer token authentication via `$ASANA_PERSONAL_ACCESS_TOKEN`") and then a Prerequisites section. We insert a new **Token Resolution** section between Prerequisites and Authentication, and update the opening line and every curl example to use `$ASANA_TOKEN` (the resolved token variable) instead of `$ASANA_PERSONAL_ACCESS_TOKEN`.

- [ ] **Step 1: Replace the opening description line**

In `plugins/asana-workflow/skills/asana-api/SKILL.md`, change line 11:

Old:
```
Common Asana REST API patterns for task management workflows. All operations use bearer token authentication via `$ASANA_PERSONAL_ACCESS_TOKEN`.
```

New:
```
Common Asana REST API patterns for task management workflows. All operations use bearer token authentication via a resolved token (see Token Resolution below).
```

- [ ] **Step 2: Replace the Prerequisites section**

Old:
```markdown
## Prerequisites

- `$ASANA_PERSONAL_ACCESS_TOKEN` env var — Asana personal access token
  - If missing: guide user to https://app.asana.com/0/my-apps
  - Add to `~/.zshrc`: `export ASANA_PERSONAL_ACCESS_TOKEN="your-token"`
```

New:
```markdown
## Prerequisites

- `$ASANA_PERSONAL_ACCESS_TOKEN` env var — primary Asana personal access token (required)
  - If missing: guide user to https://app.asana.com/0/my-apps
  - Add to `~/.zshrc`: `export ASANA_PERSONAL_ACCESS_TOKEN="your-token"`
- Additional tokens (optional) — stored as `ASANA_TOKEN_<NAME>` env vars, e.g.:
  - `export ASANA_TOKEN_WORK="your-work-token"`
  - `export ASANA_TOKEN_CLIENT_X="your-client-x-token"`
```

- [ ] **Step 3: Insert the Token Resolution section after Prerequisites and before Authentication**

Insert this block between the Prerequisites section and the `## Authentication` heading:

```markdown
## Token Resolution

At the start of every invocation, resolve which token to use and treat it as `$ASANA_TOKEN` for all subsequent API calls in this skill.

**Resolution order:**

1. **Check conversation context** — if a token override was set earlier in this session (e.g., user said "use my work account"), use that token value directly.
2. **Otherwise** — use `$ASANA_PERSONAL_ACCESS_TOKEN` as the default.

**Switching accounts (conversational):**

When the user says something like "use my work Asana account", "switch to the client token", or any similar intent:

1. Run `env | grep ^ASANA_TOKEN_` to discover available named tokens.
2. Match the user's phrasing conversationally against the discovered names (e.g., "work" → `ASANA_TOKEN_WORK`, "client x" → `ASANA_TOKEN_CLIENT_X`).
3. If exactly one match: set it as the active token override in conversation context. Confirm: "Switched to ASANA_TOKEN_WORK for this session."
4. If multiple plausible matches: list the options and ask the user which to use.
5. If no match found: report clearly (e.g., "No ASANA_TOKEN_* var found matching 'work'. Available: ASANA_TOKEN_CLIENT_X") and fall back to the default.

**Error handling for the resolved token:**

- If the resolved var is not set in the environment: report it and fall back to `$ASANA_PERSONAL_ACCESS_TOKEN`.
- If an API call returns 401 on the switched token: report "ASANA_TOKEN_WORK appears invalid or expired (HTTP 401)." and offer to fall back to the default.

The active token override is session-only — nothing is written to disk.
```

- [ ] **Step 4: Update the Authentication section to use `$ASANA_TOKEN`**

Old:
```markdown
## Authentication

All requests include:

```
Authorization: Bearer $ASANA_PERSONAL_ACCESS_TOKEN
```
```

New:
```markdown
## Authentication

All requests use the token resolved above:

```
Authorization: Bearer $ASANA_TOKEN
```
```

- [ ] **Step 5: Replace all `$ASANA_PERSONAL_ACCESS_TOKEN` occurrences in curl examples**

In the Common Operations section, every curl command references `$ASANA_PERSONAL_ACCESS_TOKEN`. Replace all occurrences with `$ASANA_TOKEN`.

There are 7 occurrences across the following curl commands:
- Fetch Task Details
- Move Task to Section (list sections)
- Move Task to Section (addTask)
- Update Custom Field
- Post Comment on Task
- Fetch Subtasks
- Fetch Task Stories

For each, change `-H "Authorization: Bearer $ASANA_PERSONAL_ACCESS_TOKEN"` to `-H "Authorization: Bearer $ASANA_TOKEN"`.

- [ ] **Step 6: Verify the file looks correct**

Read through `plugins/asana-workflow/skills/asana-api/SKILL.md` and confirm:
- Token Resolution section appears between Prerequisites and Authentication
- No remaining `$ASANA_PERSONAL_ACCESS_TOKEN` in curl examples (only in Prerequisites where it's documented as the env var name)
- Authentication section references `$ASANA_TOKEN`

- [ ] **Step 7: Commit**

```bash
git add plugins/asana-workflow/skills/asana-api/SKILL.md
git commit -m "feat: add multi-token resolution to asana-api skill"
```

---

### Task 3: Update `setup.sh` — add optional additional accounts loop

**Files:**
- Modify: `setup.sh`

The addition goes after the closing `fi` of the primary token block (after line 203) and before the Step 4 comment block. It is a self-contained bash block.

- [ ] **Step 1: Insert the additional accounts block after the primary token step**

In `setup.sh`, find the line:

```bash
# ─────────────────────────────────────────────
# Step 4: GitHub token for auto-updates
# ─────────────────────────────────────────────
```

Insert the following block immediately before it:

```bash
# Optional: additional Asana accounts
if [ -n "${ASANA_PERSONAL_ACCESS_TOKEN:-}" ]; then
  echo ""
  read -rp "  Add additional Asana accounts (ASANA_TOKEN_<NAME>)? [y/N] " ADD_MORE
  ADD_MORE=${ADD_MORE:-N}
  if [[ "$ADD_MORE" =~ ^[Yy]$ ]]; then
    while true; do
      echo ""
      read -rp "  Account name (e.g. 'work', 'client_x') — leave blank to stop: " ACCT_NAME
      [ -z "$ACCT_NAME" ] && break

      # Uppercase and prefix
      ACCT_VAR="ASANA_TOKEN_$(echo "$ACCT_NAME" | tr '[:lower:]' '[:upper:]' | tr ' ' '_')"

      if exists_in_profile "$ACCT_VAR"; then
        warn "${ACCT_VAR} already exists in ${PROFILE} — skipping"
        continue
      fi

      read -rp "  Paste token for ${ACCT_VAR}: " ACCT_TOKEN_INPUT
      if [ -z "$ACCT_TOKEN_INPUT" ]; then
        warn "No token provided — skipping ${ACCT_VAR}"
        continue
      fi

      add_to_profile "$ACCT_VAR" "$ACCT_TOKEN_INPUT"

      # Validate
      ACCT_HTTP=$(curl -s -o /dev/null -w "%{http_code}" \
        -H "Authorization: Bearer $ACCT_TOKEN_INPUT" \
        "https://app.asana.com/api/1.0/users/me")

      if [ "$ACCT_HTTP" = "200" ]; then
        ACCT_USER=$(curl -s -H "Authorization: Bearer $ACCT_TOKEN_INPUT" \
          "https://app.asana.com/api/1.0/users/me?opt_fields=name,email" \
          | python3 -c "import sys,json; d=json.load(sys.stdin)['data']; print(f\"{d['name']} ({d['email']})\")" 2>/dev/null || echo "unknown")
        pass "${ACCT_VAR} valid — ${ACCT_USER}"
      elif [ "$ACCT_HTTP" = "401" ]; then
        warn "${ACCT_VAR} token is invalid or expired (HTTP 401)"
        info "Regenerate at: https://app.asana.com/0/my-apps"
      else
        warn "Asana API returned HTTP ${ACCT_HTTP} for ${ACCT_VAR} — could not verify"
      fi
    done
  fi
fi

```

- [ ] **Step 2: Verify the block is correctly placed**

Run a quick sanity check — the file should still be valid bash:

```bash
bash -n setup.sh
```

Expected output: (no output, exit code 0)

- [ ] **Step 3: Commit**

```bash
git add setup.sh
git commit -m "feat: add optional additional Asana accounts to setup.sh"
```

---

### Task 4: Smoke test the skill change manually

**Files:** No changes — verification only

- [ ] **Step 1: Confirm no `$ASANA_PERSONAL_ACCESS_TOKEN` in curl examples**

```bash
grep -n "ASANA_PERSONAL_ACCESS_TOKEN" plugins/asana-workflow/skills/asana-api/SKILL.md
```

Expected: only matches in the Prerequisites and Token Resolution sections (documenting the env var name), none inside backtick curl command blocks.

- [ ] **Step 2: Confirm `$ASANA_TOKEN` appears in the Authentication section and all curl examples**

```bash
grep -n "ASANA_TOKEN" plugins/asana-workflow/skills/asana-api/SKILL.md
```

Expected: lines in Prerequisites (ASANA_TOKEN_WORK example), Token Resolution section, Authentication section, and all 7 curl command blocks.

---

### Task 5: Final commit and branch ready for PR

**Files:** No changes — git only

- [ ] **Step 1: Confirm all changes are committed**

```bash
git status
```

Expected: `nothing to commit, working tree clean`

- [ ] **Step 2: Verify commit history on the branch**

```bash
git log main..HEAD --oneline
```

Expected: 2 commits — one for the skill, one for setup.sh (plus the spec commit if it landed on this branch).
