---
name: upstream-breakout
description: Select, claim, and implement an independent Juju 4.1 upstream task in a dedicated worktree, then prepare its branch and PR for human review. Use for the next upstream breakout or to start a specific task from UPSTREAM-PLAN.md.
---

# Upstream breakout

The planning repository is three directories above this skill's directory.
Resolve its documents there even when the session runs from the parent
directory containing the Juju worktrees. Read `UPSTREAM-PLAN.md`, the selected
task's full acceptance criteria, and the target repository's instructions.
Use [references/validation.md](references/validation.md) for validation
and dispatch details. Consult `UPSTREAM-FIXES.md`, `UPSTREAM-DONE.md`, and PoC
code only when needed for the assigned task.

## Select and reserve work

1. If the user names a task, use it only if it is available and its
   prerequisites are met. Otherwise find the highest-priority ready,
   unclaimed item. Consider open reviews and existing implementations; do
   not recreate them or start a task whose behavior or dependencies are
   unresolved. When priority, scope, or behavior requires a decision, explain
   the evidence and ask the user before choosing or implementing.
2. Coordinate skill-based changes to the shared planning documents with the
   lock described in `UPSTREAM-PLAN.md`. Under the lock, re-read the plan and
   claim table and confirm the candidate has not been claimed, started, or
   superseded. Record the task, owner/session, intended branch and worktree,
   status, and next action in the claim table before creating the worktree.
   Release the lock promptly. Never hold it while waiting for the user, running
   tests, fetching from GitHub, or implementing.
3. Check current upstream 4.1 code, GitHub PRs, branch and worktree names, and
   the exact prerequisite state. Refresh and record the upstream base commit.
   If a conflict, duplicate, unavailable prerequisite, or unexpected
   dependency appears, update the claim to blocked under the lock and ask
   the user; do not appropriate another owner's checkout.
4. Create the dedicated worktree and branch from the verified base, except
   for an explicitly documented stack. Use the existing shared `_deps` as
   directed by the plan. Record the actual base and worktree in the claim
   under the lock. A claimed task is not permission to reset, switch, rebase,
   or otherwise alter another worktree.

## Implement and prepare review

- Implement one review concern using the target branch's APIs and tests.
  Respect the task's paths, behavior contract, regression cases and the
  working agreement in the plan. Ask the user about substantive ambiguities;
  mark the claim blocked while waiting. Do not silently widen scope.
- Validate the actual change using the reference guide and the task's own
  requirements. Report failed, skipped, and unavailable checks honestly.
  Coordinate resource-intensive tests with the user when needed.
- Record progress and blockers in the claim table under the lock, including
  after worktree creation, a blocker, and completion. Retain the claim while
  waiting for input; do not treat a paused session as abandoned. Only release
  or reassign a claim with the owner's or user's explicit authorization.
- When ready, create a conventional commit with the required co-author
  trailer. Present the exact branch, destination, and commit before pushing.
  A push is not PR publication and does not replace review of the PR text.
- Prepare the exact PR title, base, head, body, and create action locally.
  Follow the target repository's PR guidance: write a prospective merge
  message with reproducible functional QA steps, not review history or a
  completed-test log. Show it to the user and obtain explicit approval of
  that content and action before creating the PR, including a draft PR.
  If the user requests changes, show the revised content for approval.
- Once published with approval, record the PR link and current status in the
  plan and finding register under the lock. Report the worktree, branch,
  commit, validation, PR link, and remaining review/CI/QA actions. Do not
  merge, modify published PR content, or resolve review threads without
  separate explicit approval.
