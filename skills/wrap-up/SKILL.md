---
name: wrap-up
description: Use when ending a working session in the daily-work harness — to commit the session, mark a sub-task or phase done, merge a finished sub-task to main, or save a handoff before stepping away.
---

# wrap-up

Closes a session. See `${CLAUDE_PLUGIN_ROOT}/daily-workflow.md` for the loop this fits.

**Invariant: all `_dev/` edits (TODO ticks, the phase archive, handoff docs) are committed on `main`** — never on a sub-task branch. When code work happened in a worktree, make those `_dev/` edits in the main checkout.

## Steps

1. **Commit the session.** Conventional commits, as many as the work needs:
   - Code changes → on the current branch (the worktree sub-task branch, or `main` for spec work). Scope: `<type>(phase-<N>): …`.
   - `_dev/` changes → on `main`. Scope: `<type>(phase-<N>): …` for phase/sub-task work, bare `<type>: …` off-phase.

2. **Branch on session outcome:**

   **Done — spec written:** the phase's spec is complete and its `N.M` sub-tasks are inlined into `_dev/TODO.md`. The commit (step 1) is all that's needed — the phase stays `[WIP]` (sub-tasks remain) and is now a spec'd phase. **No handoff doc.**

   **Done — sub-task complete:**
   - Tick its `N.M` checkbox in `_dev/TODO.md` (on `main`).
   - If it was the **last** sub-task of the phase, ask the user whether to mark `## Phase N` `[done]`. On yes, mark it and **archive the phase**:
     - Move the phase's whole section — from its `## Phase N` heading to the line before the next `## ` heading — **verbatim** to the end of `_dev/TODO-done.md` (create the file with a `# <project> — Dev TODO, done phases` title the first time). No rewording, no trimming.
     - Leave one index line in `_dev/TODO.md` under `## Done — Phases <first>–<N>`, a section above the first open phase, created the first time with the sentence ``Each phase's full section, its sub-tasks and what landed, as it closed: `_dev/TODO-done.md`.`` Update the range in its heading. The line: ``- **Phase N — <title>** · done <YYYY-MM-DD> · spec `<the phase's spec path>` ``.
   - **Ask the user to confirm the merge.** Only on confirmation: merge the sub-task branch (`phase-<N>/<M>-<kebab>`) into `main`, then remove its worktree. Never merge unprompted.

   **Mid-session** — work genuinely unfinished (spec or sub-task):
   - Overwrite the handoff doc (on `main`), kept short: where things stand, what's next, any open thread.
     - Spec work → `_dev/docs/handoff/phase-<N>.md`
     - Sub-task work → `_dev/docs/handoff/task-<N.M>.md`
   - One per scope, overwritten each wrap-up — do not create a second file.

## Notes

- The handoff doc is what `daily-work-harness:pick-up-task` reads on a `[WIP]` resume — keep it scannable, not a transcript.
- Don't tick a checkbox or mark a phase `[done]` for work that isn't actually finished.
- The archive is what keeps `_dev/TODO.md` small enough to read whole at every pickup; a closed phase's record lives on in `_dev/TODO-done.md`, unchanged.
