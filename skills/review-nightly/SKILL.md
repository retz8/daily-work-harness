---
name: review-nightly
description: Use when reviewing the pull requests an autonomous run left on the remote — a morning inbox of open `phase-<N>/<M>-*` sub-task PRs to triage, decide on, and merge back into the daily-work harness. Also when the user invokes `/review-nightly`.
---

# review-nightly

Triages the sub-task PRs an autonomous run left on the remote and merges the good ones back into the harness. See `${CLAUDE_PLUGIN_ROOT}/daily-workflow.md` for the loop this closes. Run from the repo root (`main`).

**Invariant: this is the merge path for an autonomous sub-task PR** — the remote-PR counterpart to `daily-work-harness:wrap-up`'s local-worktree merge. The merge lands the branch's plan doc + code on `main`; the `_dev/TODO.md` tick that follows is committed on `main`.

## Steps

1. **Build the inbox.** List open PRs whose head branch matches `phase-<N>/<M>-*` (`gh pr list --state open --json number,headRefName,title,isDraft,createdAt`). Present a compact queue — one row per PR: number, sub-task `N.M`, title, and whether it is a `[needs-attention]` draft (a run that failed its gates) or a green merge candidate. If none, say so and stop.

2. **Pick one.** User chooses; offer "oldest first" to walk the queue in order. Never auto-pick, never auto-merge.

3. **Build the review card** for the chosen PR, without leaving the terminal:
   - the sub-task's line in `_dev/TODO.md` and its phase spec `_dev/docs/spec/phase-<N>-*.md`,
   - the plan doc on the branch (`_dev/docs/plan/task-<N.M>-*.md`),
   - the diff (`gh pr diff <n>`),
   - the green-gate block from the PR body.

4. **Review for correctness, anchored to the spec.** Hand the diff to `code-review` (or a subagent) with the phase spec as the bar. Judge three things specifically:
   - Does the change actually fulfill `N.M` as the spec defines it?
   - Real bugs in the diff?
   - Scope creep — did it touch `_dev/` beyond its own plan doc, or another sub-task's surface?

   Surface the findings. Don't decide for the user.

5. **Decide** (ask the user):
   - **Merge** → step 6.
   - **Request changes** — post the findings on the PR (`code-review --comment`) and leave it open. This sub-task is now yours to finish by hand: while the PR stays open the routine skips it and will not iterate on it.
   - **Verify locally first** — `gh pr checkout <n>` to run the app or gates yourself, then return to this menu. Re-running gates locally is opt-in; the routine already ran them in the cloud.
   - **Close** — reject: `gh pr close <n> --delete-branch`.

6. **Merge and reconcile**, only on the user's go-ahead:
   - `gh pr merge <n> --merge --delete-branch` (merge commit, matching the harness's worktree merges).
   - `git checkout main && git pull`.
   - Hand to `daily-work-harness:wrap-up` for the tick. The branch is already merged and deleted, so wrap-up's merge clause no-ops; it ticks `N.M` in `_dev/TODO.md` on `main` and, if this was the phase's last sub-task, asks whether to mark `## Phase N` `[done]`.
   - Return to the inbox (step 1) for the next PR.

## Notes

- One open PR per sub-task is the routine's guarantee — the inbox never shows duplicates, so each row is a distinct decision.
- `[needs-attention]` drafts are failed runs, not merge candidates: review the failure, then fix by hand or close. Never merge a draft as-is.
- Never merge without the user's explicit go-ahead.
