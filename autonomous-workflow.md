# Autonomous workflow

The map of the autonomous half of the daily-work harness: a delegated task becomes a GitHub issue, a nightly routine produces a gated PR, and `review-nightly` merges it back. The **Contract** section below is the coupling all three pieces share; the **Produce** section maps the nightly routine. Kept concise — its readers read it by section.

## Contract

The issue/label contract the writer (`delegate-task`), the producer (the nightly routine), and the closer (`review-nightly`) all conform to.

### The unit

A delegated task is one GitHub **issue** = one PR = one merge, created in the project's own `origin` repo. Two kinds:

- **`kind:spec`** — the definition is a committed spec + a `_dev/TODO.md` `N.M` line; the producer reads the spec.
- **`kind:standalone`** — self-contained; the full definition is in the issue body.

### Labels

Issue and PR labels never overlap: **issue labels = queue / classification / dependency; PR labels = run outcome.**

Issue labels:

| Label | Meaning |
|---|---|
| `autonomous-ready` | the selector — the human's "safe to run unattended" gate; the routine's filter |
| `kind:spec` \| `kind:standalone` | how to read the task |
| `blocked-by:#<N>` (0+) | one per unmet dependency; auto-cleared when the dependency closes |
| `blocked:setup` | the routine tried and could not start (missing skill / path / malformed issue); human-cleared |

PR labels — every produced PR carries **exactly one**:

| Label | Meaning | review-nightly |
|---|---|---|
| `review-ready` | produced clean, ready for merge | case-1 |
| `needs-input` | run stuck on a decision it can't make | case-2 |
| `needs-attention` | run failed mechanically (gates red / errored) | fix or close |

`autonomous-ready` (entry) and `review-ready` (exit) are the paired gates: the human gates entry, the machine gates exit.

### States

Most states are inferred, not labelled:

- **queued** — open issue, `autonomous-ready`, no linked PR, no `blocked-by:*` remaining, not `blocked:setup`.
- **blocked** — carries ≥1 `blocked-by:#<N>`, or `blocked:setup`; the routine skips it.
- **produced** — an open PR exists; its outcome label (`review-ready` / `needs-input` / `needs-attention`) says whether it's mergeable, blocked on input, or failed. Produced PRs are never drafts — the label alone gates merge, and any open PR stops re-grabbing (the one-open-PR-per-task guarantee).
- **done** — issue closed (its PR merged).

### Dependencies — `blocked-by:#<N>`

Dependencies are `blocked-by:#<N>` labels, one per blocking issue; unblocked = zero remain. When a dependency issue closes, its `blocked-by:#<N>` label is deleted repo-wide, which strips it from every dependent at once. Ordering follows: a dependency is delegated first so its number exists to reference.

### Issue schema

Title:

- `kind:spec` → `[N.M] <sub-task title>`
- `kind:standalone` → an imperative title

Body:

| Section | Content |
|---|---|
| **Task** | `kind:standalone`: the full definition. `kind:spec`: "Phase N sub-task N.M". |
| **References** | `kind:spec` only — the spec that defines the sub-task: the task-level `task-<N.M>` spec if one exists, else the phase spec (+ plan-doc path if it exists). By path, never embedded. |
| **Plan skill** | present only when planning is warranted; names the skill to plan first |
| **Execution skill** | present only when a specific skill should drive the work; else the routine runs freeform |
| **Branch** | `phase-<N>/<M>-<kebab>` (spec) \| `task/<kebab>` (standalone) |
| **Depends on** | the blocking issue numbers, mirroring `blocked-by:#<N>`; omitted if none |

The issue body is the **sole authority** on which skills run — the routine invokes exactly the named skills, freeform if none. The PR auto-wires `Closes #<N>` so a merge closes the issue.
