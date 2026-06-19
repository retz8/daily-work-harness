---
name: rebase-with-main
description: Rebase the current worktree's sub-task branch onto main to absorb anything that landed. Use when the user invokes `/rebase-with-main`, when resuming a sub-task, or after work lands on main while a sub-task worktree is in flight.
---

# rebase-with-main

See `${CLAUDE_PLUGIN_ROOT}/daily-workflow.md` for how this fits the daily loop. Sub-task work happens in worktrees off `main` (`phase-<N>/<M>-<kebab>`); this skill keeps the active branch current.

## Workflow

1. **Guard.** Current branch must be a sub-task branch (`phase-<N>/<M>-<kebab>`) — never `main`. Working tree must be clean; if not, stop and ask.
2. `git fetch origin main`
3. `git rebase origin/main`
4. **On conflict:** stop immediately, show the conflicting files, and let the user decide — never auto-resolve, never `--abort` without asking.
5. **If the branch is already pushed:** `git push --force-with-lease`. Never force-push `main`.
6. Report how many commits were replayed and whether `main` had moved at all (a no-op is a valid outcome).
