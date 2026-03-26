# Multi-Asana-Token Support — Design Spec

**Date:** 2026-03-26
**Status:** Approved

## Problem

The user has two Asana accounts and therefore two different personal access tokens. The current system supports exactly one token via `$ASANA_PERSONAL_ACCESS_TOKEN`. There is no way to switch accounts without manually changing the env var.

## Goal

Support multiple named Asana tokens with a conversational switching mechanism and a sensible default.

## Scope

### In scope

- Token resolution logic in `asana-api` skill
- Optional additional accounts step in `setup.sh`

### Out of scope

- Persisting the active token across sessions
- Per-project token config files
- Auto-detecting the account from the task URL or workspace

---

## Design

### Token naming convention

Additional tokens are stored as env vars following the pattern `ASANA_TOKEN_<NAME>` in `~/.zshrc` (or `~/.bashrc`). Examples:

```
ASANA_PERSONAL_ACCESS_TOKEN=...   # primary / default
ASANA_TOKEN_WORK=...
ASANA_TOKEN_CLIENT_X=...
```

### Token resolution — `asana-api` skill

A **Token Resolution** section is added at the top of `asana-api/SKILL.md`, before the Authentication section. It defines the following logic, applied at the start of every `asana-api` invocation:

1. **Check conversation context** for an active token override set earlier in the session. If found, use it.
2. **Otherwise**, use `$ASANA_PERSONAL_ACCESS_TOKEN` as the default.
3. **When the user requests a switch** (e.g., "use my work Asana account", "switch to the client token"):
   - Run `env | grep ^ASANA_TOKEN_` to discover available named tokens
   - Match the user's intent conversationally against the discovered names (e.g., "work" → `ASANA_TOKEN_WORK`)
   - Store the resolved token as the active override in conversation context for the rest of the session
   - Confirm to the user: "Switched to ASANA_TOKEN_WORK for this session."
4. **If the intent is ambiguous** and multiple `ASANA_TOKEN_*` vars exist, list them and ask which to use.
5. **If no matching var is found**, report clearly and fall back to the default.

The active token override is a conversation-context concept only — nothing is written to disk.

### `setup.sh` changes

After the primary token step (Step 3), add an optional loop for additional accounts:

1. After the primary token is verified, prompt: "Do you want to add any additional Asana accounts? [y/N]"
2. If yes, loop:
   - Prompt for a name (e.g., `work`) — uppercased to form `ASANA_TOKEN_WORK`
   - Prompt for the token value
   - Write to profile via `add_to_profile` (deduplication already handled)
   - Validate via `/users/me` API call, same as the primary token
   - Ask: "Add another account? [y/N]"
3. If no (or after the loop), continue to Step 4 as before.

Re-running `setup.sh` later is safe — `exists_in_profile` skips vars that are already set.

---

## Files Changed

| File | Change |
|------|--------|
| `plugins/asana-workflow/skills/asana-api/SKILL.md` | Add Token Resolution section |
| `setup.sh` | Add optional additional accounts loop after Step 3 |

## Files Unchanged

`start-task`, `ship-it`, `create-pr`, `pre-ship-check`, `work-summary`, all `references/` files.

---

## Error Handling

- Unknown account name → report and list available `ASANA_TOKEN_*` vars
- Referenced env var not set → report and fall back to default
- API 401 on switched token → report invalid/expired, offer to fall back to default
