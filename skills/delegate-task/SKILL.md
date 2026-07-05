---
name: delegate-task
description: Project a task — a spec'd `N.M` sub-task, or a standalone hotfix/chore — into a GitHub issue conforming to the autonomous-run contract, for the nightly routine to pick up. Use when the user invokes `/delegate-task` or asks to delegate / hand off a task to the autonomous routine.
---

# delegate-task

Projects a task into a GitHub issue the nightly routine can run unattended. **Read `${CLAUDE_PLUGIN_ROOT}/autonomous-workflow.md` in full first** — it defines the issue/label contract this skill writes. Interactive and human-gated; run from the repo root (`main`). Issues are created in the project's own `origin` repo.

## Workflow

1. **Resolve target + kind.**
   - Invoked with `N.M` → **`kind:spec`**. Read its `_dev/TODO.md` line (assume it is already `[WIP]`, picked up) + the spec that defines the sub-task (the task-level `_dev/docs/spec/task-<N.M>-*.md` if one exists, else the phase spec), plus `_dev/docs/plan/task-<N.M>-*.md` if present.
   - Invoked with no argument or a free-form description → **`kind:standalone`**. Interview for the full definition; derive a kebab title.

2. **Readiness gate.** Confirm the task is defined enough to run unattended.
   - `kind:spec`: the phase spec must exist — if not, stop ("spec the phase first"). If the definition is only a thin phase-spec handle, surface the choice: grill / `superpowers:writing-plans` **now, this session**, or proceed trusting the Plan-skill section to let the routine plan from the phase spec. "No plan doc" is **not** "not ready"; "not ready" is a definition too thin for even planning to land — kick that back to a grill/spec.
   - `kind:standalone`: the interview captures the definition; a large one may reference a spec/plan path instead of inlining it.
   - This gate normally passes trivially — `/delegate-task` is usually invoked right after a grill/spec in the same session.

3. **Dedupe.** List open issues. Refuse if the task already has one — one open issue per task (`kind:spec` keys on the `[N.M]` title / the `(#N)` in the TODO line; `kind:standalone` cannot auto-dedupe, so use judgment).

4. **Judge skills — propose, the human confirms or overrides.**
   - **Plan:** plan doc exists → none, execute it. No plan doc and planning **not** warranted (small, well-scoped) → propose **direct execution**, no Plan-skill section. No plan doc and planning warranted (multi-step, needs decomposition) → propose a **Plan skill**. Bias toward the leaner option.
   - **Execution:** propose the execution skill that fits the project (read its `CLAUDE.md` + installed skills), or none for freeform.

5. **Dependencies.** Ask for the blocking issue numbers; ensure each `blocked-by:#<N>` label exists and apply it. A dependency must already be delegated so its number exists.

6. **Preview — the human gate.** Assemble the full issue (title, body per the contract's schema, labels) and show it. **Nothing hits GitHub until the user approves.**

7. **Create.** Ensure the issue-side labels exist (`autonomous-ready`, the `kind:*`, any `blocked-by:#<N>`), then create the issue with them in `origin`.

8. **Wire TODO** (`kind:spec` only): append `(#<new>)` to the sub-task's `[WIP]` line in `_dev/TODO.md` on `main`.

9. **Report** the issue URL, then stop.

## Notes

- The label vocabulary, issue schema, and states live in `autonomous-workflow.md` — this skill **writes** them, it does not redefine them.
- `/delegate-task` only creates the issue. The routine produces the PR; `daily-work-harness:review-nightly` closes the issue + PR.
- Project-agnostic: skill names, spec paths, and the target repo all come from the current project — never hardcode.
