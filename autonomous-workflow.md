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
| `blocked:setup` | the routine hit a setup blocker — a missing skill / path, a malformed issue, or an environment that can't run the gates; human-cleared |

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

## Produce

The nightly routine is a self-contained prompt registered as a Claude cloud Routine via `/schedule`, run against a fresh clone of the project repo with **no harness plugin present**. The executable is [`nightly-routine-prompt.md`](./nightly-routine-prompt.md); this section is the map it is kept consistent with.

Each fire **drains the queue**: it lists every **eligible** issue — open, `autonomous-ready`, no `blocked-by:*`, no `blocked:setup`, no open PR — oldest first, then works them one at a time, **a subagent per issue**. Each issue is finished (commit → push → open PR) before the next starts, so a completed task's open PR is a durable checkpoint: the run is resumable across fires, and unreached issues roll to the next fire.

### Outcomes

Running one issue ends in exactly one terminal outcome:

| Outcome | Artifact | Label | Re-grabbed? |
|---|---|---|---|
| clean | non-draft PR | `review-ready` (PR) | no — open PR |
| blocked on a decision | non-draft PR, partial work | `needs-input` (PR) | no — open PR |
| gates red / mid-run error | non-draft PR | `needs-attention` (PR) | no — open PR |
| precondition failure | issue comment, no PR | `blocked:setup` (issue) | no — label excludes it |
| hard crash | none | none | yes, next fire |

A **precondition failure** is a setup blocker the routine can't fix itself — a named skill not committed to the clone, a referenced spec/plan path absent, an unparseable issue body, or an environment that cannot run the project's gates. The routine applies `blocked:setup` and leaves one comment (blocker + fix), then stops without a PR:

```
## ⛔ blocked:setup — autonomous run could not start
- **Blocker:** <one line: which precondition failed>
- **Details:** <the skill name / missing path / absent field>
- **Fix:** <what to do, then remove the `blocked:setup` label to re-queue>
```

### PR shape

Every produced PR is **non-draft**, its title mirrors the issue, and its body carries a `## Related issue` section with `Closes #<N>` (a merge closes the issue), a summary, a gates block, and one per-outcome section: recorded assumptions (`review-ready`), a **Decision needed** writeup (`needs-input`), or a **Failure** writeup (`needs-attention`). Exactly one outcome label is applied.

### Routine setup

Per-project operator setup, done once when adopting the routine (ungraded):

- Register `nightly-routine-prompt.md`'s content via `/schedule` as a **Remote** routine (not a local/desktop task — Remote runs fully autonomously with no approval prompts), on a nightly cadence, on a capable model (e.g. Opus).
- Enable **unrestricted branch pushes** for the repo — Routines default to `claude/`-prefixed branches, but the contract uses `phase-<N>/<M>-*` / `task/<kebab>`.
- Ensure the environment provides the **base toolchain and network** the gates need (language runtimes, allowed domains) — the routine installs the project's own dependencies from its documented setup.
- Ensure the project's Plan/Execution skills are committed under `.claude/skills/` — the routine has no access to plugin skills, only repo-committed ones.
