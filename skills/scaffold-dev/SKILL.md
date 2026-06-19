---
name: scaffold-dev
description: Use when a repository has no `_dev/` working area yet and the daily-work harness needs its tracking structure created — e.g. a fresh repo where `daily-work-harness:pick-up-task` finds no `_dev/TODO.md`.
---

# scaffold-dev

## Overview

Creates the `_dev/` working area the harness reads and writes. One-shot and **idempotent — never clobbers an existing `_dev/`**. See `${CLAUDE_PLUGIN_ROOT}/daily-workflow.md` for the model these artifacts serve.

## When to use

- A fresh repo with no `_dev/` directory.
- `daily-work-harness:pick-up-task` reports there is no `_dev/TODO.md` to read.

Do **not** use when `_dev/` already exists — stop and report instead of overwriting.

## What it creates

```
_dev/
  TODO.md
  docs/
    spec/.gitkeep
    plan/.gitkeep
    handoff/.gitkeep
```

## Steps

1. **Guard.** If `_dev/` already exists, stop and report — make no changes.
2. Create the tree above. The `.gitkeep` files keep the empty `docs/` subdirs tracked.
3. Write `_dev/TODO.md` with the skeleton below.
4. Report what was created, and tell the user to add their first `## Phase 1` then run `daily-work-harness:pick-up-task`.

## TODO.md skeleton

```md
# Dev TODO

## Phase 1 — <title>
<one-line goal of the phase>

## Backlog
```

## Notes

- `_dev/` lives on `main` — never scaffold it on a worktree/sub-task branch.
- This skill only creates the structure; phases and their sub-tasks are authored by the user (and the spec track), not generated here.
