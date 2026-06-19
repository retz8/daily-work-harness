# daily-work-harness

A Claude Code plugin: a general daily-work harness — phases → sub-tasks → worktrees → specs → handoffs, driven from a repo's `_dev/` directory. Project- and domain-agnostic.

> _Description TBD._

## Prerequisite

The spec/sub-task flow hands off to **`grill-me`** (a standalone skill). This plugin assumes `grill-me` is installed and available in your environment; it is not bundled.

## Install

Declaratively, in a consuming repo's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "retz8-harness": {
      "source": { "source": "github", "repo": "retz8/daily-work-harness" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": { "daily-work-harness@retz8-harness": true }
}
```

Or interactively:

```
/plugin marketplace add retz8/daily-work-harness
/plugin install daily-work-harness@retz8-harness
```

## Skills

Invoked namespaced as `/daily-work-harness:<skill>`:

- **pick-up-task** — session dispatcher; reads `_dev/TODO.md`, routes to spec or sub-task work.
- **grill-to-spec** — convert a completed `grill-me` session into a spec under `_dev/docs/spec/`.
- **rebase-with-main** — rebase a sub-task worktree branch onto `main`.
- **wrap-up** — close a session: commit, tick `_dev/TODO.md`, merge a finished sub-task, or save a handoff.

The workflow model these fit is documented in [`daily-workflow.md`](./daily-workflow.md).
