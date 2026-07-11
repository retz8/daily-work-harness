# Autonomous workflow

The map of the autonomous half of the daily-work harness: a delegated task becomes a GitHub issue, a nightly routine produces a gated PR, and `review-nightly` merges it back. The **Contract** section below is the coupling all three pieces share; **Produce** maps the nightly routine; **Review** maps the morning triage. Kept concise — its readers read it by section.

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

PR labels — mutually exclusive; a produced PR carries **exactly one** outcome label, which `review-nightly` may later swap for `autonomous-revise-ready` to hand the PR back:

| Label | Meaning | review-nightly |
|---|---|---|
| `review-ready` | produced clean, ready for merge | case-1 |
| `needs-input` | run stuck on a decision it can't make | case-2 |
| `needs-attention` | run failed mechanically (gates red / errored) | fix or close |
| `autonomous-revise-ready` | a human resolved the blocker; the routine may resume this branch | set by `review-nightly` (re-delegate) |

`autonomous-ready` (entry) and `review-ready` (exit) are the paired gates: the human gates entry, the machine gates exit.

### States

Most states are inferred, not labelled:

- **queued** — open issue, `autonomous-ready`, no linked PR, no `blocked-by:*` remaining, not `blocked:setup`.
- **blocked** — carries ≥1 `blocked-by:#<N>`, or `blocked:setup`; the routine skips it.
- **produced** — an open PR exists; its outcome label (`review-ready` / `needs-input` / `needs-attention`) says whether it's mergeable, blocked on input, or failed. Produced PRs are never drafts — the label alone gates merge, and any open PR stops re-grabbing (the one-open-PR-per-task guarantee).
- **re-queued for revision** — an open PR relabelled `autonomous-revise-ready` by `review-nightly` after a human resolved its blocker; the routine re-grabs it and resumes on the existing branch.
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

Each fire **drains the queue** from two sources: every **eligible issue** — open, `autonomous-ready`, no `blocked-by:*`, no `blocked:setup`, no open PR — and every **`autonomous-revise-ready` PR** (one whose blocker a human resolved via `review-nightly`). Work-items are ordered oldest first and worked one at a time, **a subagent per item**. A fresh issue produces a new PR on a new branch; a revise PR is **resumed on its existing branch** — the subagent re-reads the issue's resolution comment and, for `kind:spec`, the updated spec on that branch, continues the work, re-runs gates, and flips `autonomous-revise-ready` to the resulting outcome label (`review-ready` on a clean pass). Each item is finished (commit → push → PR) before the next starts, so an open PR is a durable checkpoint: the run is resumable across fires, and unreached items roll to the next fire.

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

On the **clean path only** (gates green), the routine also runs a **mechanical self-review** of its own diff against the bar before opening the PR, and records the findings in a `## Mechanical review` section (or a "no findings" note if clean). These findings are **advisory** — they do not change the outcome: `review-ready` stays `review-ready` regardless of what the self-review surfaces, since only red *gates* produce `needs-attention`. The morning `review-nightly` reads this section instead of driving a mechanical pass live.

### Routine setup

Per-project operator setup, done once when adopting the routine (ungraded):

- Register `nightly-routine-prompt.md`'s content via `/schedule` as a **Remote** routine (not a local/desktop task — Remote runs fully autonomously with no approval prompts), on a nightly cadence, on a capable model (e.g. Opus).
- Enable **unrestricted branch pushes** for the repo — Routines default to `claude/`-prefixed branches, but the contract uses `phase-<N>/<M>-*` / `task/<kebab>`.
- Ensure the environment provides the **base toolchain and network** the gates need (language runtimes, allowed domains) — the routine installs the project's own dependencies from its documented setup.
- Ensure the project's Plan/Execution skills are committed under `.claude/skills/` — the routine has no access to plugin skills, only repo-committed ones.
- Create the labels the routine applies — `blocked:setup`, `review-ready`, `needs-input`, `needs-attention` — on the repo once. The routine assumes they exist and cannot create them (no label-create tool, direct REST blocked); a missing one is silently skipped, losing the queue/outcome signal.

## Review

`review-nightly` is the morning triage/merge counterpart, run locally from `main`. It builds a four-lane inbox from **labels, not branches**, and drives one item at a time to a terminal state; nothing merges or acts without the human's go-ahead.

### The four lanes

Two queries — open PRs by outcome label, open issues by `blocked:setup`:

| Lane | Source | Disposition |
|---|---|---|
| `review-ready` | PR | review → merge (case-1) |
| `needs-input` | PR | resolve the blocker → fix or re-delegate (case-2) |
| `needs-attention` | PR | fix or reject — never merge as-is |
| `blocked:setup` | issue (no PR) | report the blocker → re-queue once setup is fixed |

### Reviewing a PR

The chosen PR is reviewed on a **plain local checkout** of its head branch in the repo root — one PR at a time, so no isolation is needed: mechanical mode reads only `gh pr diff`, local mode runs the gates / app on the checkout. The checkout is **rebased onto `main` locally before review** (no push — the remote PR is untouched until a terminal decision) so the diff and any local-mode gates are judged against what it will merge into; a rebase conflict halts for the human. Mechanical review is **pre-computed** — a `review-ready` PR already carries a `## Mechanical review` section the routine wrote at night, so the default path is to **read that section** and **run local review** (the project's gates / app on the branch); driving a fresh `code-review` pass live is an opt-in re-run, worth it only to judge the diff against the rebased `main` (the night reviewed against `main`-as-of-production). The bar is the task-level `task-<N.M>` spec for `kind:spec` (phase spec as context), the issue body for `kind:standalone`. It judges: does it fulfil the bar, real bugs, scope creep (`_dev/` beyond its own plan doc, or another task's surface). Findings are surfaced; the human decides. A **worktree** is set up only when a path needs to write to the branch (a live fix, or a case-2 spec edit): `git checkout main` to release the branch, then `git worktree add` off the head branch as the isolated fix channel, removed on the terminal action.

### Decision menu

- **Merge** (`review-ready`, post-review) — merge commit, delete branch; `Closes #<N>` closes the issue. `kind:spec` hands to `wrap-up` for the `_dev/TODO.md` tick (and, if it was the phase's last sub-task, the `[done]` prompt); `kind:standalone` has no tick.
- **Fix live** — set up the fix channel (release the branch to `main`, `git worktree add` off the head branch), apply the fix there, push (the PR updates in place), re-review, merge.
- **Re-delegate** — relabel the PR `autonomous-revise-ready` and leave it open; the routine resumes it next fire (see Produce).
- **Reject** — close the PR and delete its branch; leave the issue open with `autonomous-ready` stripped.

A `review-ready` PR that fails review drops into the same fix / re-delegate / reject fork. `needs-attention` uses the same fork with **no** spec-change precursor.

### Case-2 — resolving a `needs-input` blocker

The PR carries a **Decision needed** writeup. The human clarifies it (escalating to a grill if deep), then records the resolution **on the PR branch, not `main`**: for `kind:spec` the governing spec is edited in the fix-channel worktree — creating a `task-<N.M>` spec if only a phase-spec handle existed — and reaches `main` only at the eventual merge; for `kind:standalone` nothing on the branch changes. Either way the resolution is written as an **issue comment** in three parts — progress so far / why the previous attempt was wrong / the resolved decision — and, for `kind:spec`, a **PR comment** noting what was resolved alongside the spec change. Then the fork: fix live, or re-delegate.

### `blocked:setup`

Report-only: `review-nightly` surfaces the routine's blocker comment; the operator fixes the project setup (outside the harness). Once fixed, it offers to **re-queue** — strip `blocked:setup` and ensure `autonomous-ready` — or the human rejects, closing the issue.
