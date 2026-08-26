---
name: scaffold-claude-md
description: Use after `daily-work-harness:scaffold-dev` on a repo adopting the harness, to append the harness's operating conventions to the repo's `CLAUDE.md` — e.g. `_dev/` now exists but `CLAUDE.md` says nothing about how the harness works.
---

# scaffold-claude-md

## Overview

Appends the harness's operating conventions to the adopting repo's `CLAUDE.md`, so a session picks up the rules the skills rely on without being told. One-shot and **idempotent — never rewrites conventions already present**.

The block appended is `${CLAUDE_PLUGIN_ROOT}/claude-md-setup.md`, verbatim.

## When to use

- Right after `daily-work-harness:scaffold-dev` has created `_dev/`.
- A repo already using `_dev/` whose `CLAUDE.md` carries no harness conventions.

Do **not** use to refresh an already-present block. If the conventions are stale, delete the `### Daily-work harness` section by hand and re-run.

## Steps

1. **Guard.** If the repo-root `CLAUDE.md` already contains a `### Daily-work harness` heading, stop and report — make no changes.
2. Read the template at `${CLAUDE_PLUGIN_ROOT}/claude-md-setup.md`.
3. Append it to `./CLAUDE.md` at the repo root, creating the file if it does not exist. If that file already has a `## Conventions` heading, append only the `### Daily-work harness` subsection under it — do not add a second `## Conventions`.
4. Report what was added and to which file.

## Notes

- The target is the repo-root `CLAUDE.md` — the one checked into the repo, not the user's global `~/.claude/CLAUDE.md`.
- `CLAUDE.md` lives on `main`, like `_dev/` — never append on a worktree/sub-task branch.
- The block states only what holds for any adopter. Repo-specific conventions (sibling repos, design-record layouts, a named nightly Routine) are authored by the adopter alongside it, not generated here.
