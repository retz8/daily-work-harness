# Autonomous workflow

The map of the autonomous half of the daily-work harness: a delegated task becomes a GitHub issue, a nightly routine produces a gated PR, and `review-nightly` merges it back. `daily-work-harness:delegate-task` reads this doc in full, so it stays concise. The produce and review sections land with the 5.2 / 5.3 sub-tasks.

## The unit

A delegated task is one GitHub **issue** = one PR = one merge, created in the project's own `origin` repo. Two kinds:

- **`kind:spec`** — the definition is a committed spec + a `_dev/TODO.md` `N.M` line; the producer reads the spec.
- **`kind:standalone`** — self-contained; the full definition is in the issue body.

## Labels

Issue and PR labels never overlap: **issue labels = queue / classification / dependency; PR labels = run outcome.**

Issue labels:

| Label | Meaning |
|---|---|
| `autonomous-ready` | the selector — the human's "safe to run unattended" gate; the routine's filter |
| `kind:spec` \| `kind:standalone` | how to read the task |
| `blocked-by:#<N>` (0+) | one per unmet dependency |

PR labels — every produced PR carries **exactly one**:

| Label | Meaning | review-nightly |
|---|---|---|
| `review-ready` | produced clean, ready for merge | case-1 |
| `needs-input` | run stuck on a decision it can't make | case-2 |
| `needs-attention` | run failed mechanically (gates red / errored) | fix or close |

`autonomous-ready` (entry) and `review-ready` (exit) are the paired gates: the human gates entry, the machine gates exit.

## States

Most states are inferred, not labelled:

- **queued** — open issue, `autonomous-ready`, no linked PR, no `blocked-by:*` remaining.
- **blocked** — carries ≥1 `blocked-by:#<N>`; the routine skips it.
- **in-progress / produced** — an open non-draft PR exists (also the one-open-PR-per-task signal that stops re-grabbing).
- **done** — issue closed (its PR merged).

### `blocked-by:#<N>`

Dependencies are `blocked-by:#<N>` labels, one per blocking issue; unblocked = zero remain. When a dependency issue closes, its `blocked-by:#<N>` label is deleted repo-wide, which strips it from every dependent at once. Ordering follows: a dependency is delegated first so its number exists to reference.

## Issue schema

Title:

- `kind:spec` → `[N.M] <sub-task title>`
- `kind:standalone` → an imperative title

Body:

| Section | Content |
|---|---|
| **Task** | `kind:standalone`: the full definition. `kind:spec`: "Phase N sub-task N.M". |
| **References** | `kind:spec` only — the spec path (+ plan-doc path if it exists). By path, never embedded. |
| **Plan skill** | present only when planning is warranted; names the skill to plan first |
| **Execution skill** | present only when a specific skill should drive the work; else the routine runs freeform |
| **Branch** | `phase-<N>/<M>-<kebab>` (spec) \| `task/<kebab>` (standalone) |
| **Depends on** | the blocking issue numbers, mirroring `blocked-by:#<N>`; omitted if none |

The issue body is the **sole authority** on which skills run — the routine invokes exactly the named skills, freeform if none. The PR auto-wires `Closes #<N>` so a merge closes the issue.
