---
description: Commit current changes with type prefix (TASK/BUGFIX/BREAKING)
allowed-tools: Bash
---

Create a commit with the specified type prefix.

## Arguments
- `$ARGUMENTS` - Optional: "bugfix" or "breaking" (default: "task")

## Instructions

1. Run `git status` to see changes
2. Run `git diff` and `git diff --staged` to understand what changed
3. Stage relevant files (prefer specific files over `git add -A`)
4. Determine commit type from argument:
   - "bugfix" or "fix" → `[BUGFIX]`
   - "breaking" → `[BREAKING]`
   - anything else (including empty) → `[TASK]`
5. Extract ticket number from branch name if present (e.g., `feature/ABC-817-...` → `#ABC-817`)
6. Check if GPG signing is available: `git config --get user.signingkey`
   - If key exists: use `-S` flag
   - If no key: commit without `-S`
7. Create commit with format:
   ```
   [TYPE] #TICKET: Brief description
   ```
   **Do NOT add Co-Authored-By or any other trailer.**
8. Show result with `git log -1`