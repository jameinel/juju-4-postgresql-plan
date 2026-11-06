---
name: upstream-daily-summary
description: Summarize the Juju PostgreSQL upstream plan daily, check open PR reviews and CI, record newly merged branches, and identify unstarted work. Use when asked for a daily upstream plan, branch, or PR status summary.
---

# Daily upstream summary

The planning repository is three directories above this skill's directory.
Resolve its files relative to that repository, not the current working
directory; the session may run from a parent containing multiple worktrees.
Use its `UPSTREAM-PLAN.md` as the queue, not a fixed list of PRs.
`UPSTREAM-DONE.md` records implementation and merge history;
`UPSTREAM-FIXES.md` records finding status and PoC reconciliation. Consult
those files as needed to interpret or update the queue.

Preserve the plan's SQL/schema root-cause acceptance contract when updating
tasks. A boundary-only PoC workaround is not a completed upstream schema
correction; report the distinction without making PostgreSQL support or
execution a new requirement for stock 4.1 work.

## Gather current state

1. Read the plan and inventory every proposed branch, open PR, unstarted task,
   dependency and reconciliation obligation, including items outside its
   main open-reviews table. Deduplicate entries.
2. For work already recorded as merged, rely on the plan or completed-work
   record; do not probe the PR just to reconfirm its status. Summarize it
   briefly and retain any pending integration work.
3. Check GitHub for every PR the plan still treats as open. Prefer `gh` and
   inspect the current state, head, base, mergeability, review decision,
   individual checks, review summaries, issue comments, and review threads
   including replies and resolved or outdated threads where relevant. Check
   pagination rather than assuming the first page is complete. Compare
   feedback with later commits and author responses; distinguish addressed
   concerns, unanswered warnings, optional suggestions, and approval requests.
   An empty unresolved-thread list does not mean there is no feedback in a
   review body.
4. For failing or canceled checks, identify the affected job and, when useful,
   inspect its steps and logs. Report specific symptoms and links without
   claiming a root cause that the evidence does not establish. Check the
   aggregate job's dependencies or logs before attributing its failure.
   Skipped checks are not passes; pending or unavailable results are unknown.
5. Ignore the macOS client job as a task to investigate. If its cancellation
   alone causes the aggregate check to fail, say so once and note that GitHub
   may still block merging. Do not ignore other failures or assume an
   aggregate failure is macOS-related without checking.
6. Preserve stack ordering, prerequisites, functional QA, and post-merge
   reconciliation separately from review and CI status. For unstarted items,
   distinguish unclaimed upstream branches from work already implemented in
   a PoC, and distinguish ready-to-start tasks from candidates needing scope.
   Consult the plan's breakout claim list so reserved, active, or blocked
   tasks are not reported as unstarted.
   If current state cannot be verified, label it as the plan's recorded state
   rather than asserting it is unstarted everywhere.

## Record newly merged work

When a PR the plan identifies as open has merged, update the local planning
documents as part of this workflow. Confirm its merge date, merge commit, and
final head from GitHub first. Re-read the affected files before updating them
and preserve other local changes.

- Move its implementation and review history out of the open-review queue and
  into the landed section of `UPSTREAM-DONE.md`. Preserve useful rationale and
  historical validation, with dates and heads; do not imply old checks cover a
  later head.
- Keep a concise reference to the branch in `UPSTREAM-PLAN.md` when follow-up
  remains, particularly PoC reconciliation. Record the actual merged state,
  acceptance contract, and next action there; do not leave it as an open PR.
- Update matching current-status entries in `UPSTREAM-FIXES.md` so they do not
  contradict the merge. Preserve historical snapshots as historical, and do
  not mark PoC integration, QA, or PostgreSQL validation complete without
  evidence.
- Respect existing local changes and worktree ownership. Do not reset, rebase,
  commit, publish, modify GitHub content, or resolve review threads as part of
  this workflow. If a local change conflicts or merge details cannot be
  verified, report the blocker instead of guessing.

## Report

Lead with the most important changes and actionable blockers. Give:

- A short merged-work summary, highlighting new merges and remaining
  integration obligations.
- A linked entry for each unmerged PR, with the branch or concern and the
  specific action on the user's side: review response, approval, CI
  investigation, functional QA, stack dependency, or none beyond waiting.
  Separate verified failures from hypotheses and optional review suggestions.
- A list of unstarted upstream branches and other follow-ups, identifying
  prerequisites and PoC work already done.

State the time or date of the GitHub snapshot. If GitHub access or pagination
fails, report what could not be checked and do not present stale data as live.
Mention any planning-document updates. Keep the report concise without
omitting an actionable review comment or non-macOS CI failure.
