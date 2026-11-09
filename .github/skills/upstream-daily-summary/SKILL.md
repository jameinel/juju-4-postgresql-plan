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
   Classify linked PRs as user-owned work, external dependencies, or
   contextual references. Use the plan's assignments/claims and GitHub author
   to establish ownership; a link alone does not make a PR part of the user's
   work queue. Bot-authored work can be user-owned when the plan assigns it.
   For another author's PR, do not attribute feedback to the user unless a
   comment, review request or explicit assignment directs an action to them.
   If ownership or the intended respondent is unclear, report that uncertainty
   rather than inventing a response obligation.
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
   An unresolved thread is not necessarily unanswered: check later PR-level
   replies and commits too. Distinguish "implementation needed", "implemented,
   response needed", and "answered, awaiting reviewer confirmation". An
   outdated thread alone does not establish that its concern was addressed.
   Classify each reviewer by `user.type` and login. Human reviewers come
   first: their comments, questions, change requests and approvals determine
   merge readiness, and an unanswered human comment is the highest-priority
   review action. Van Cleef (`lee-van-cleef[bot]`) is an important automated
   reviewer; record its verdict and the count and substance of its issues
   and suggestions, and whether each has been answered or addressed since.
   Its verdict is not a human approval. Other bots are lower priority unless
   they block merging. Do not present bot findings as established defects;
   they still need evaluation with the user (see the
   `prepare-pr-review-response` skill).
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

Organize the report by **next action and next actor**, not PR number, branch
age, or whether a review exists. Start with the GitHub snapshot time and
counts of merge-ready PRs, PRs needing user action, and PRs waiting on others.
Count each queued PR once; keep unpublished work and backlog counts separate.
Exclude contextual references and other authors' dependencies from these
counts unless the user has an explicitly assigned action on that PR.

Use the following sections in order. Keep "Ready to merge" and "Human
responses needed" visible even when empty; omit other empty sections.

1. **Ready to merge.** Only include PRs whose current head satisfies required
   human review and CI, has no unresolved required feedback, merge hold,
   unmet prerequisite or outstanding acceptance QA, and is mergeable.
   A historical human approval, bot verdict, or absence of failing checks
   is insufficient. Skipped or unavailable checks do not establish required
   validation. State the next action as obtaining the user's explicit merge
   authorization, not performing a merge as part of this summary.
2. **Human responses needed - your action.** Unanswered human questions,
   change requests and actionable comments come first among review work.
   Say whether code, a decision, or just a response is still needed. Do not
   put an already answered concern here merely because its thread remains
   unresolved. An approval with accompanying requests still needs evaluation.
3. **Bot feedback to evaluate - your action.** List unanswered Van Cleef
   issues before optional suggestions, with verdict and outstanding counts.
   Describe findings as claims to evaluate, not confirmed defects. Explicit
   acceptance, reasoned rejection or deferral can complete the response;
   optional suggestions do not automatically become merge blockers.
   Addressed historical findings do not keep a PR in this section.
4. **Other work you can advance.** CI investigation, functional QA, semantic
   decisions/rework, draft preparation and currently actionable stack
   maintenance. Name the concrete next step. Keep explicit merge holds
   prominent, even when checks pass. Group repeated CI symptoms into one
   investigation with links to every affected PR, without assuming a shared
   root cause. Do not suggest a rebase before its prerequisite lands.
5. **Waiting on others - no action now.** Awaiting human review/re-review,
   reviewer confirmation, running CI, a parent PR, or an external dependency.
   Name who or what must act and the event that makes the PR actionable again.
   Distinguish existing human approval with an unmet GitHub review gate from
   no human review yet. Use "request/re-request review" in an action section
   only when evidence supports a missing request; otherwise say "await review".
   A draft is not awaiting ordinary review unless it was explicitly requested.
6. **Unpublished work, backlog and integration.** Keep claimed active/blocked
   work separate from unclaimed ready-to-start tasks and candidates needing
   scope. Identify prerequisites and existing PoC implementations. Briefly
   record newly merged work and remaining reconciliation obligations here;
   do not repeat the full completed-work inventory. Mention contextual PRs
   only when useful, label them as context, and do not give them a response
   action. Show external dependencies as blockers on the user's affected
   branch rather than as additional user-owned review entries.

Give each unmerged PR in the user's work queue one primary entry, in the
first applicable action section; use the waiting section only when no user
action can advance it.
Include other actionable blockers in that entry rather than duplicating the
PR across sections. Bot feedback must not hide a verified CI failure or
merge hold. Identify each PR by its full head branch name in code formatting
and a linked PR number, together in the same column. Do not replace branch
names with PR numbers or truncate them. Omit redundant PR titles and owner
prefixes on branch names unless needed to disambiguate.

Each populated PR section MUST use a Markdown table with these columns,
even when it contains only one PR. Section headings alone, prose entries and
bullet lists are not substitutes. For a required empty section, write "None."
without an empty table. Keep cells concise; put necessary extended evidence
after the table, keyed to its branch/PR.

| Branch / PR | Next action | Waiting on / other blockers |
|---|---|---|

Within each entry, report human review state first, then outstanding Van
Cleef feedback and relevant other bots. Summarize answered bot findings as
answered rather than reproducing their old counts as outstanding work.
Preserve pending functional QA, stack order and reconciliation separately
from review/CI status. Avoid vague instructions such as "finish review" or
"monitor CI" when the evidence supports a more specific action or wait state.

Within action sections, prioritize blocking requests and work that unlocks
other branches, then the oldest unanswered feedback. Show the feedback date
or waiting-since date when known so neglected reviews are easy to spot.
Call out changes since a previous summary only when that baseline is known.

If GitHub access or pagination fails, identify what could not be checked and
label the affected classification provisional; do not infer merge readiness
or "no action needed" from missing evidence. If a head changes during the
snapshot, refresh its review/CI classification before reporting it.
Mention planning-document updates. Keep the report concise without omitting
an actionable human comment or non-macOS CI failure. This workflow does not
publish responses, request reviews, resolve threads, or merge PRs.

Before returning, verify that every populated PR section has a table, every
queued PR has its branch name and linked number, counts match those entries,
and no other author's feedback has been assigned to the user merely because
the plan links their PR.
