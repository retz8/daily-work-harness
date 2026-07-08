---
name: delegate-task
description: Project a task — a spec'd `N.M` sub-task, or a standalone hotfix/chore — into a GitHub issue conforming to the autonomous-run contract, for the nightly routine to pick up. Use when the user invokes `/delegate-task` or asks to delegate / hand off a task to the autonomous routine.
---

# delegate-task

Projects a task into a GitHub issue the nightly routine can run unattended. **First read the contract** — `sed -n '/^## Contract/,/^## Produce/p' ${CLAUDE_PLUGIN_ROOT}/autonomous-workflow.md` — the issue/label contract this skill writes. Interactive and human-gated; run from the repo root (`main`). Issues are created in the project's own `origin` repo.

## Workflow

1. **Resolve target + kind.**
   - Invoked with `N.M` → **`kind:spec`**. Read its `_dev/TODO.md` line (assume it is already `[WIP]`, picked up) + the spec that defines the sub-task (the task-level `_dev/docs/spec/task-<N.M>-*.md` if one exists, else the phase spec), plus `_dev/docs/plan/task-<N.M>-*.md` if present.
   - Invoked with no argument or a free-form description → **`kind:standalone`**. Interview for the full definition; derive a kebab title.

2. **Readiness gate.** Confirm the task is defined enough to run unattended.
   - `kind:spec`: the phase must be spec'd (else stop — "spec the phase first"). If the sub-task is only a thin phase-spec handle, offer to grill / `superpowers:writing-plans` now, or proceed and let the Plan skill plan from the phase spec. A missing plan doc is **not** a blocker; too thin to plan from is — send that back to a grill/spec.
   - `kind:standalone`: the interview captures the definition; a large one may point to a spec/plan path instead of inlining.
   - Normally trivial — `/delegate-task` usually follows a grill/spec in the same session.

3. **Dedupe.** List open issues. Refuse if the task already has one — one open issue per task (`kind:spec` keys on the `[N.M]` title / the `(#N)` in the TODO line; `kind:standalone` cannot auto-dedupe, so use judgment).

4. **Judge skills — propose, the human confirms or overrides.**
   - **Plan:** plan doc exists → execute it, no Plan skill. No plan doc → propose **direct execution** for a small/well-scoped task, or a **Plan skill** if it needs decomposition. Bias toward the leaner option.
   - **Execution:** propose the execution skill that fits the project (read its `CLAUDE.md` + installed skills), or none for freeform.
   - **Name skills by their bare committed `.claude/skills/<name>/` dir name — never a `superpowers:`/plugin prefix.** The routine runs in a fresh clone with no plugins; a prefixed name won't resolve and fails the item to `blocked:setup`.

5. **Dependencies.** Ask for the blocking issue numbers; ensure each `blocked-by:#<N>` label exists and apply it. A dependency must already be delegated so its number exists.

6. **Preview — the human gate.** Assemble the full issue (title, body per the contract's schema, labels) and show it. **Nothing hits GitHub until the user approves.**

7. **Create.** Ensure the issue-side labels exist (`autonomous-ready`, the `kind:*`, any `blocked-by:#<N>`), then create the issue with them in `origin`.

8. **Wire TODO** (`kind:spec` only): append `(#<new>)` to the sub-task's `[WIP]` line in `_dev/TODO.md` on `main`.

9. **Commit + push to `main`.** The routine runs against a fresh clone and reads the referenced spec from it — an uncommitted spec is an absent path (`blocked:setup`). Stage only the `_dev/` files this delegation touches — the spec doc the issue references (and the plan doc if it references one), plus the `_dev/TODO.md` wiring from step 8 — never a blanket add of unrelated working-tree changes. Commit with a `docs` conventional-commit message and push to `origin main`. If the issue embeds its whole definition inline and references no doc (a `kind:standalone` case), there is nothing to commit — skip.

10. **Report** the issue URL, then stop.

## Notes

- The label vocabulary, issue schema, and states live in `autonomous-workflow.md` — this skill **writes** them, it does not redefine them.
- `/delegate-task` only creates the issue. The routine produces the PR; `daily-work-harness:review-nightly` closes the issue + PR.
- Project-agnostic: skill names, spec paths, and the target repo all come from the current project — never hardcode.
