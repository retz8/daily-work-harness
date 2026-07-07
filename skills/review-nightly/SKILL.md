---
name: review-nightly
description: Use when reviewing what an autonomous run left on the remote — a morning inbox of labelled PRs (`review-ready` / `needs-input` / `needs-attention`) and `blocked:setup` issues to triage, review on their live branch, and drive to merge / fix / re-delegate / reject. Also when the user invokes `/review-nightly`.
---

# review-nightly

The morning triage/merge counterpart to the nightly producing routine: build a four-lane inbox from the labels a run left behind, review one item at a time on its live branch, and drive each to a terminal state. Run from the repo root (`main`).

**First read the contract + the review map** — the two sections of `autonomous-workflow.md` this skill closes against:

- Contract: `sed -n '/^## Contract/,/^## Produce/p' ${CLAUDE_PLUGIN_ROOT}/autonomous-workflow.md`
- Review: `sed -n '/^## Review/,$p' ${CLAUDE_PLUGIN_ROOT}/autonomous-workflow.md`

**Invariant: this is the merge/close path for an autonomous PR** — the remote counterpart to `daily-work-harness:wrap-up`'s local-worktree merge. A merge lands the branch's code and whatever `_dev/` it carried (a plan doc, or a case-2 spec change) on `main`; the `_dev/TODO.md` tick that follows is committed on `main`. Never merge or act without the user's explicit go-ahead; never auto-pick.

## Steps

1. **Build the inbox.** Two label-keyed queries (not branch globs, so `kind:spec` and `kind:standalone` are covered uniformly):
   - `gh pr list --state open --json number,headRefName,title,labels,createdAt` → the three outcome lanes (`review-ready`, `needs-input`, `needs-attention`).
   - `gh issue list --state open --label blocked:setup` → the no-PR lane.

   Present a **clear four-lane list**, oldest-first within each lane — one row per item: number, `[N.M]` / standalone title, kind, lane. If every lane is empty, say so and stop.

2. **Pick one.** The user chooses; offer "oldest first" to walk a lane. Never auto-pick, never auto-act. A `blocked:setup` row → step 7. A PR row → step 3.

3. **Check out the live PR branch.** `git fetch origin <headRef>` then `git checkout <headRef>` in the repo root — a **plain local checkout** is the review substrate (one PR at a time, so no isolation is needed). Mechanical mode reads only `gh pr diff`; local mode runs the gates / app on this checkout. **No worktree at review time** — the fix channel is set up later, only if the review sends you into the fix fork (step 6). On the terminal action you return to `main`.

4. **Review.** Offer three modes and run the chosen one:
   - **(1) mechanical** — hand the diff + the bar to `code-review` (or a review subagent);
   - **(2) local** — run the project's documented gates / app on the checked-out branch;
   - **(3) both** — mechanical first, then local.

   The **bar** is by kind: `kind:spec` → the task-level `_dev/docs/spec/task-<N.M>-*.md` (primary; the phase spec is context), `kind:standalone` → the issue body. Mechanical review judges three things: does the change **fulfil the bar**, **real bugs** in the diff, **scope creep** (touched `_dev/` beyond its own plan doc, or another task's surface). Surface findings — never decide for the user.

5. **Decide** (ask the user), by lane:
   - **`review-ready`** → **Merge** (step 6a). If the review fails, it drops into the fix fork (step 6b/6c).
   - **`needs-input`** → **case-2**: resolve the blocker (step 6b-precursor) first, then the fork.
   - **`needs-attention`** → the fix fork (step 6b/6c) with **no** spec change.
   - Any PR → **Reject** (step 6d).

6. **Drive to terminal state** (only on go-ahead). Merge / reject / re-delegate act on the PR directly. Any path that **writes to the PR branch** — the case-2 spec edit (6b-precursor) or a live fix (6b) — first sets up the **fix channel**: `git checkout main` in the repo root to release the branch (review had it checked out), then `git worktree add <path> <headRef>` off the PR's head branch. That worktree is the isolated fix substrate, removed on the terminal action.

   **6a — Merge.** `gh pr merge <n> --merge --delete-branch` (`Closes #<N>` auto-closes the issue). Remove the fix worktree if one was created, then `git checkout main && git pull`. For **`kind:spec`**, hand to `daily-work-harness:wrap-up` for the `N.M` tick on `main` (and, if it was the phase's last sub-task, the `[done]` prompt). For **`kind:standalone`**, no tick. Return to the inbox.

   **6b-precursor — case-2 resolution (only for `needs-input`).** Read the PR's **Decision needed** writeup. Clarify it with the user — escalate to `grill-me` if the decision is deep/multi-branch, otherwise a lightweight Q&A. Record the resolution **on the PR branch, not `main`** (set up the fix channel per the step-6 preamble first):
   - `kind:spec` → in the fix-channel worktree, edit the governing spec (create `_dev/docs/spec/task-<N.M>-*.md` if only a phase-spec handle existed), commit on the branch. It reaches `main` only at the eventual merge.
   - Leave an **issue comment** in three parts — *progress so far / why the previous attempt was wrong / the resolved decision* — and, for `kind:spec`, a **PR comment** noting what was resolved alongside the spec change.
   - `kind:standalone` → nothing on the branch changes; the issue comment is the decision record.

   Then the fork:

   **6b — Fix live.** In the fix-channel worktree, apply the fix, commit, and push (the PR updates in place — plain push, no rewrite). Return to step 4 to re-review on the worktree, then merge (6a).

   **6c — Re-delegate.** Push the branch if the fix channel carries commits (a case-2 spec change), then swap the PR's outcome label to `autonomous-revise-ready` (clearing `needs-input` / `needs-attention`); ensure the label exists first (idempotent create). Leave the PR open, branch preserved. Remove the fix worktree if one was created, keep the remote branch; return the repo root to `main`. The routine resumes it next fire. Return to the inbox.

   **6d — Reject.** `gh pr close <n> --delete-branch`. Leave the **issue open with `autonomous-ready` stripped** — the task may still be wanted, just not this attempt, and the routine won't re-grab it. Remove the fix worktree if one was created; return the repo root to `main`. Return to the inbox.

7. **`blocked:setup` lane.** Surface the routine's blocker comment (Blocker / Details / Fix) so the user sees what setup is broken. This skill is **not a fixer** — the fix (install a skill, provision env, correct a path) is operator work outside the harness. Once the user confirms setup is fixed, offer to **re-queue**: strip `blocked:setup` **and** ensure `autonomous-ready` is present (full queued state). If the task is misguided, reject instead — close the issue. Return to the inbox.

## Notes

- The label vocabulary, kinds, and PR shape live in `autonomous-workflow.md` (`## Contract` / `## Produce`) — this skill reads them, it does not redefine them.
- One open PR (or one `autonomous-revise-ready`) per task is the routine's guarantee — the inbox never shows duplicates, so each row is a distinct decision.
- `rebase-with-main` is untouched by the fix arm (it only adds commits and pushes). Use it separately only if `main` genuinely moved and the branch needs to absorb it.
- Project-agnostic: the repo, spec paths, gates, and skills all come from the current project's `origin` / `CLAUDE.md` / `_dev/` — never hardcoded.
