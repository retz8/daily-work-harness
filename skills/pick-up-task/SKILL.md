---
name: pick-up-task
description: Pick up or resume work from `_dev/TODO.md` — dispatches to spec work or sub-task work per the daily-work harness. Use when the user invokes `/pick-up-task` or asks to grab/resume a phase or sub-task.
---

# pick-up-task

The session's dispatcher. **Read `${CLAUDE_PLUGIN_ROOT}/daily-workflow.md` in full first** — it defines the model, the phase states, and the tracks. Run from the repo root (`main`).

## Workflow

1. **Read state.** If there is no `_dev/TODO.md`, run `daily-work-harness:scaffold-dev` first to create the workspace, then continue. Otherwise read `_dev/TODO.md` (phase headings, `[WIP]`/`[done]` markers, `N.M` checkboxes). Pick the earliest non-`[done]` phase; a `[WIP]` phase is resumed before any later phase. If the user named a phase or sub-task, use it.

2. **Present the next action, then stop.** Surface where pickup has landed — which phase, its state, and what the next action would be — and wait for the user's go-ahead before acting. **Auto-picking the phase is fine; never auto-proceed into the work.** What you surface depends on phase state:
   - **Fresh phase** (no `[WIP]`, no `_dev/docs/spec/phase-<N>-*.md`): next action is *start Phase N and write its spec* (grill → spec). Don't mark `[WIP]` yet.
   - **Spec'd phase** (spec exists, ≥1 unchecked `N.M`): read the phase spec **in full** + TODO, compute the **startable** sub-tasks (next-in-order plus any parallel-eligible per the spec), and present **all** of them — even if only one. A `[WIP]` sub-task carrying `(#N)` is **delegated** (running autonomously) — exclude it from the startable set, but list it under a **delegated / in-flight** heading so its issue/PR stays visible. **Never auto-pick a sub-task.** Let the user choose.
   - **`[WIP]` resume:** first read the matching handoff doc (`_dev/docs/handoff/phase-<N>.md` for spec work, `task-<N.M>.md` for sub-task work) and, for a sub-task, its `_dev/docs/plan/task-<N.M>-*.md` if present. Report where it stands, then continue the matching track. A delegated `[WIP]` (one carrying `(#N)`) has **no handoff doc** — it is in flight autonomously; point the user to its issue/PR (or `daily-work-harness:review-nightly`) instead of resuming it locally.

3. **Spec track** (fresh phase, on `main`): on go-ahead, mark `## Phase N` `[WIP]` on `main`, then hand to `grill-me` → `daily-work-harness:grill-to-spec` (writes `_dev/docs/spec/phase-<N>-*.md`) → inline numbered `N.M` sub-tasks into `_dev/TODO.md`, including which can run in parallel. Pickup stops; the user drives the grill.

4. **Sub-task track** (spec'd phase, after the user picks a sub-task from step 2):
   - **Mark `[WIP]`.** Once the sub-task is picked, mark its `N.M` checkbox `[WIP]` on `main`.
   - **Brief the chosen sub-task, then stop.** Give a short context summary — what the sub-task should achieve, drawn from both TODO and the phase spec. Do **not** proceed past this. Wait for the user's go-ahead.
   - **On go-ahead**, offer these routes (planning stays on `main`): (a) `grill-me` deeper, (b) `superpowers:writing-plans` → `_dev/docs/plan/task-<N.M>-*.md`, (c) Claude Code plan mode → implementation, (d) `daily-work-harness:delegate-task` → hand the sub-task to the nightly routine.
   - **Only when implementation begins**, create or resume the worktree off `main` (`phase-<N>/<M>-<kebab>`) and run `daily-work-harness:rebase-with-main`. The worktree holds code only; `_dev/` stays on `main`.

5. **Stop.** Pickup ends here; the user drives what happens next. Closing a session is `daily-work-harness:wrap-up`'s job, not this skill's.

## Notes

- `_dev/` (TODO, specs, plans, handoffs) is always edited on `main` — never on a sub-task branch.
- `[WIP]` marks active work at either level — on the `## Phase N` heading for a phase, and on its `N.M` checkbox for a sub-task. Never strip a `[WIP]` for work you are not picking up.
