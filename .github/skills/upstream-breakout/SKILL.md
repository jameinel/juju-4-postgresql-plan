---
name: upstream-breakout
description: Select, claim, and implement an independent Juju 4.1 upstream task in a dedicated worktree, then prepare its branch and PR for human review. Use for the next upstream breakout or to start a specific task from UPSTREAM-PLAN.md.
---

# Upstream breakout

The planning repository is three directories above this skill's directory.
Resolve its documents there even when the session runs from the parent
directory containing the Juju worktrees. Read the planning `AGENTS.md`,
`UPSTREAM-PLAN.md`, the selected `upstream/<branch>/status.md` and its full
acceptance criteria, and the target repository's instructions.
Use [references/validation.md](references/validation.md) for validation
and dispatch details. Consult `UPSTREAM-FIXES.md`, `UPSTREAM-DONE.md`, and PoC
code only when needed for the assigned task.

The plan's **Purpose and SQL/schema acceptance** section governs the approach:
fix upstream SQL/schema root causes rather than merely accommodating them at
Go conversion boundaries. The final implementation remains SQLite/dqlite-only;
the PostgreSQL PoC is evidence, not a required design or runtime dependency.

## Select and reserve work

1. If the user names a task, use it only if it is available and its
   prerequisites are met. Otherwise find the highest-priority ready,
   unclaimed item. Consider open reviews and existing implementations; do
   not recreate them or start a task whose behavior or dependencies are
   unresolved. Check whether the prescribed approach leaves the underlying
   SQL/schema problem intact, including instructions to avoid schema changes
   or preserve a mismatched representation. When priority, scope, behavior or
   that conflict requires a decision, explain the evidence and ask the user
   before choosing or implementing; do not silently expand the task.
2. Coordinate an exclusive reservation window with the coordinator. Re-read
   the index, claim table and task record; confirm the candidate has not
   been claimed, started or superseded. Record the task link and owner/session
   in the claim table before creating the worktree. Create its branch/task
   directory and `status.md` if needed; record scope, prerequisites, approved
   behavior and acceptance there. A candidate needing design agreement is
   not a dispatchable task. Do not assign new work without that agreement.
3. Check current upstream 4.1 code, GitHub PRs, branch and worktree names, and
   the exact prerequisite state. Refresh and record the upstream base commit.
   If a conflict, duplicate, unavailable prerequisite, or unexpected
   dependency appears, mark the branch record blocked and ask the user; do not
   appropriate another owner's checkout.
4. Create the dedicated worktree and branch from the verified base, except
   for an explicitly documented stack. Use the existing shared `_deps` as
   directed by the plan. Record the actual base and worktree in `status.md`.
   A claimed task is not permission to reset, switch, rebase, or otherwise
   alter another worktree.

## Implement and prepare review

- Implement one review concern using the target branch's APIs and tests.
  Respect the task's paths, behavior contract, regression cases and the
  working agreement in the plan. Ask the user about substantive ambiguities;
  mark `status.md` blocked while waiting. Do not silently widen scope.
- For SQL/schema tasks, explain why the changed SQL and stored types address
  the identified issue. Go formatting/parsing or SQL casts alone are not a fix
  for an incorrectly typed schema. Keep genuine legacy/API conversions at
  their compatibility boundaries, and distinguish documented driver
  workarounds from schema corrections.
- Validate the actual change using the reference guide and the task's own
  requirements, including SQLite and real dqlite for SQL/schema changes.
  Passing tests on those backends does not establish portable SQL. PostgreSQL
  execution is optional for resolving ambiguity, not an upstream dependency;
  ask the user when needed and report it separately. Report failed, skipped,
  and unavailable checks honestly. Coordinate resource-intensive tests with
  the user when needed.
- Record progress, heads, dated evidence and next actions only in the branch
  record after worktree creation, a blocker, and completion. Do not update
  the shared plan, finding register or completed summary on every change.
  Notify the coordinator of new findings or cross-branch effects, retaining
  the supporting evidence in the branch record. Retain the claim while
  waiting for input; do not treat a paused session as abandoned. Only release
  or reassign a claim with the owner's or user's explicit authorization.
- When ready, create a conventional commit with the required co-author
  trailer. Present the exact branch, destination, and commit before pushing.
  A push is not PR publication and does not replace review of the PR text.
- Planning history stays in the shared repository. Follow AGENTS.md's
  exclusive Git-write window and file-scoped staging/commit rules. Unrelated
  dirty files are expected, not permission to stage them or abandon the task.
- Prepare the exact PR title, base, head, body, and create action locally.
  Save the description as `upstream/<branch>/pr.md`; keep any replies beside
  it as `reply-<topic>.md`. Do not put new review drafts in the repository root.
  Follow the target repository's PR guidance: write a prospective merge
  message with reproducible functional QA steps, not review history or a
  completed-test log. Show it to the user and obtain explicit approval of
  that content and action before creating the PR, including a draft PR.
  If the user requests changes, show the revised content for approval.
- Once published with approval, record the PR link, verified head and next
  actions in `status.md`. Hand source/merge revisions and PoC reconciliation
  obligations to the coordinator for the shared summaries; do not restack
  the PoC or maintain its stage progress as part of this stock-upstream task.
  Report the worktree, branch,
  commit, validation, PR link, and remaining review/CI/QA actions. Do not
  merge, modify published PR content, or resolve review threads without
  separate explicit approval.
