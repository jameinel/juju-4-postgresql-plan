# Shared planning direction

This repository holds the PostgreSQL proof-of-concept plan and independent
upstream work in one project with shared Git history. Split document
ownership, not repositories. Read the selected workflow below; PoC operating
instructions do not govern stock-upstream work. Upstream builders use the
target checkout's commands, fixtures and environment, not PoC backend
selection, credentials or test-server setup.

## Entry points and document ownership

- PoC implementation: [PLAN.md](PLAN.md), the assigned stage, and only its
  named references. Keep the single PoC implementation/restack stream.
- Stock-upstream work: [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md), the assigned
  `upstream/<branch>/status.md`, and the target code repository's guidance.
- Coordination: the user is the coordinator. The
  [daily-summary skill](.github/skills/upstream-daily-summary/SKILL.md)
  refreshes shared summaries and identifies work to dispatch.
- PoC integration: [UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md).
  Upstream landing and verified PoC reconciliation are separate events.

The code repository's architectural, domain and nested instructions govern
code changes. This planning guidance does not replace them.

## Branch records and review artifacts

Use `upstream/<branch>/status.md` as the authoritative execution and handoff
record for that branch. Keep current state first: owner/session, branch,
worktree, base/head and observation date, PR, blockers and next actions.
Keep its scope, prerequisites, approved behavior, acceptance criteria,
decisions and revision-specific evidence below. Distinguish proposed,
implemented, validated, published, merged and reintegrated.

Keep `pr.md` beside `status.md` for the prepared PR description. Put replies
beside them as `reply-<reviewer-or-topic>.md`; add a date or suffix when
needed. Do not create a replies subdirectory or empty placeholder files.
Retain historical descriptions and replies as historical; moving a draft
does not make it match the live PR.

Directories for unclaimed tasks may use a stable task identifier until a
branch is agreed. A directory is not evidence that a branch/worktree exists.
Keep record paths stable after landing. Link related records rather than
duplicating their changing status or combining independent concerns.

## Reservations, progress and shared summaries

Serialize reservations with the coordinator. Re-read the shared claim list
and task record before recording the owner/link in UPSTREAM-PLAN.md and
creating or taking over a worktree. A paused or blocked task stays claimed.
Absence of a claim is not permission to modify an existing branch. Transfer
or release ownership only with the owner's or user's authorization.

Once reserved, workers update only their branch records and artifacts, not
shared summaries for every commit, test or review. Report new findings,
cross-branch blockers and changed scope to the coordinator immediately;
record the evidence locally without silently assigning work elsewhere.

The coordinator, or a session explicitly performing coordination, refreshes:

- UPSTREAM-PLAN.md: priorities, reservations, dated short summaries and
  cross-branch dependencies. Consult branch records before dispatch.
- UPSTREAM-FIXES.md: finding/provenance mappings, links to branch records,
  and verified PoC reconciliation evidence.
- UPSTREAM-DONE.md: concise completed-work summaries linking to retained
  branch records, not another execution log.
- PLAN.md and relevant PoC stages: only changes affecting PoC priorities,
  dependencies, milestones or actual integration results. Do not mirror
  every upstream review or CI update.

Refresh at handoff, landing or a requested status sweep. Timestamp snapshots
and distinguish recorded state from freshly verified GitHub facts. GitHub
is authoritative for live PR state. Do not overwrite another owner's dirty
record during a sweep; coordinate or leave a dated summary and report the
discrepancy. Historical evidence does not establish later-head acceptance.

## Shared Git history

Unrelated dirty planning files are expected during concurrent work. Preserve
them and continue within your assigned files unless they conflict with the
task. Workers may commit their own planning changes.

Coordinate a short exclusive Git-write window for staging, committing,
pulling and pushing this shared checkout. Git's per-command index lock does
not make a sequence of staging and committing atomic. Inspect the index
before staging; if another session has staged work, wait and coordinate.
Stage and commit only explicitly named, owned files. Never use `git add .`,
`git add -A`, or `git commit -a` here. Do not reset another session's index,
switch branches or change ownership to make a commit convenient.

Use conventional commits and the required co-author trailer. Before any
push or GitHub publication, show the exact content/commit, destination and
action and obtain explicit approval. Approval of a code patch or push does
not authorize a PR, description edit, reply, merge or thread resolution.

## Workflow-specific acceptance

Stock SQL/schema breakouts follow the
[upstream root-cause contract](UPSTREAM-PLAN.md#purpose-and-sqlschema-acceptance)
and [validation guide](.github/skills/upstream-breakout/references/validation.md):
correct the SQL/schema defect, preserve compatibility, and validate SQLite
and real dqlite. PostgreSQL execution is optional diagnostic evidence, not
an upstream runtime dependency or substitute for supported-backend checks.

PoC runtime, schema generation, environment, stack and operational gates
remain in PLAN.md, OPERATIONS.md and stages/. The independently scoped
PostgreSQL test-infrastructure branch has its own record; it is not a new
acceptance requirement for each SQL/schema breakout. Ask before changing
scope or turning an unapproved design into implementation.
