# Daily-work harness

The map of how work flows from `_dev/TODO.md` through a working session. `pick-up-task` reads this doc in full to dispatch, so it stays concise. Single author.

## Model

- The **phase** (`## Phase N` in `_dev/TODO.md`) is the unit of work. **Sub-tasks** (`N.M`) are its implementation steps, derived after the phase spec is written.
- **All of `_dev/` lives on `main`** — `TODO.md`, `docs/spec/`, `docs/plan/`, `docs/handoff/` are edited and committed on `main` only.
- **Code lives in worktrees off `main`.** A worktree branch carries only implementation code, never `_dev/` edits.

## Dispatch — `pick-up-task`

Reads `_dev/TODO.md` + this doc, then routes on phase state. Phase selection: the earliest non-`[done]` phase; a `[WIP]` phase is resumed before any later phase.

| Phase state | Route |
|---|---|
| **Fresh** — no `[WIP]`, no `_dev/docs/spec/phase-<N>-*.md` | Mark `## Phase N` `[WIP]`. → **Spec track** (on `main`). |
| **Spec'd** — spec exists, ≥1 unchecked `N.M` | → **Sub-task track**. |
| **`[WIP]` resume** | First read the matching handoff doc (`phase-<N>.md` for spec work, `task-<N.M>.md` for sub-task work) and, for a sub-task, the plan doc if present. Then route as above. |

## Spec track (on `main`)

`grill-me` → `grill-to-spec` (writes `_dev/docs/spec/phase-<N>-*.md`) → inline numbered `N.M` sub-tasks into `_dev/TODO.md` under `## Phase N`. The spec and decomposition **identify which sub-tasks can run in parallel** vs. which are ordered or blocked.

## Sub-task track

1. Read the phase spec **in full** + TODO; compute the **startable** sub-tasks (next-in-order plus any parallel-eligible). **Present all of them — even if only one. Never auto-pick.** User chooses.
2. Present a summary of the chosen sub-task, drawn from the phase spec.
3. Offer three choices — **planning happens on `main` first**:
   - `grill-me` deeper into the sub-task.
   - `superpowers:writing-plans` → plan to `_dev/docs/plan/task-<N.M>-*.md`.
   - Claude Code plan mode → straight to implementation.
4. **Only when implementation begins**, create the worktree off `main` (`phase-<N>/<M>-<kebab>`, via `rebase-with-main`). It holds only code.

## Wrap-up — `wrap-up`

- Commit the session's changes (conventional commits; multiple commits fine).
- If the sub-task is done: tick `N.M`; if it was the last, ask to mark the phase `[done]`. Owns the user-permitted worktree → `main` merge.
- If invoked mid-session: overwrite the concise handoff doc (`_dev/docs/handoff/phase-<N>.md` or `task-<N.M>.md`).

## Autonomous PRs — `review-nightly`

An autonomous run can produce a sub-task PR off `main` on the same `phase-<N>/<M>-<kebab>` branch — plan doc + code on the branch, gates run before the PR opens. `review-nightly` is the morning counterpart: triage the open `phase-<N>/<M>-*` PRs, review each against its phase spec, and on go-ahead merge (merge commit) and hand to `wrap-up` for the `N.M` tick. One open PR per sub-task; `[needs-attention]` drafts are failed runs.

## Branching, merge, commits

- Worktrees off `main`, one per sub-task: `phase-<N>/<M>-<kebab>`. `rebase-with-main` absorbs anything that landed on `main`. Merge to `main` is user-permitted, via `wrap-up`.
- Conventional commits: `<type>(phase-<N>): …` for phase/sub-task work; bare `<type>: …` off-phase.

## Artifacts (all tracked, on `main`)

- `_dev/docs/spec/phase-<N>-*.md` — phase spec (kept).
- `_dev/docs/plan/task-<N.M>-*.md` — optional sub-task plan (kept).
- `_dev/docs/handoff/{phase-<N>,task-<N.M>}.md` — concise session summary, one per scope, overwritten each `wrap-up`.

## Skills

`daily-work-harness:scaffold-dev` (one-time `_dev/` setup), `daily-work-harness:pick-up-task` (dispatcher), `daily-work-harness:wrap-up`, `daily-work-harness:rebase-with-main`, `daily-work-harness:review-nightly` (autonomous-PR triage), plus `grill-me` / `daily-work-harness:grill-to-spec`.
