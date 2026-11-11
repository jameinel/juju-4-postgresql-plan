# 4.1-cte-membership-subqueries

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Head `67daa565a6`: gfouillet approved; his follow-ups were implemented and answered, awaiting confirmation. Investigate revision smoke and aggregate failures; validate the follow-up on real dqlite and complete manual QA. Preserve explicit single-column membership and scoped cleanup, lifecycle and departure semantics.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-PLAN.md`:

[4.1-cte-membership-subqueries](https://github.com/jameinel/juju/tree/4.1-cte-membership-subqueries), [#23750](https://github.com/juju/juju/pull/23750) | `67daa565a6`: review follow-ups to the eleven membership expressions and departure-scope regression | Reviewer confirmation, real-dqlite validation, CI triage and manual QA remain.

From `UPSTREAM-DONE.md`:

`4.1-cte-membership-subqueries`, [#23750](https://github.com/juju/juju/pull/23750) | X24 | Replace eleven SQLite membership shorthands with explicit single-column subqueries; preserve filtering, scoped cleanup, lifecycle and departure semantics. Rebased head `02e9f73e49` retains the same patch.

### Upstream register snapshot

`4.1-cte-membership-subqueries` | X24 | [#23750](https://github.com/juju/juju/pull/23750) | `67daa565a6` | open; human follow-ups answered; revision smoke and aggregate failed; real-dqlite follow-up validation and manual QA remain

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

## Existing CTE membership implementation — X24

**PR published 2026-11-04:** continue `4.1-cte-membership-subqueries` in
juju/juju#23750; do not dispatch a duplicate X24 implementation.
The original implementation started from upstream/4.1 base
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` and replaced eleven
production expressions in eight files (plus focused regressions):
- `domain/application/state/{application_endpoint,k8sservice}.go`
- `domain/removal/state/model/{application,unit,storage}.go`
- `domain/operation/state/{task,prune}.go`
- `domain/relation/state/relation.go`

#23693 touches `prune.go` for size expressions, not CTE membership; keep
this independent and preserve that review's separate ownership (possible
same-file integration conflict).
Contracts: preserve refresh endpoint filtering, orphan/shared hostname and
pod-device deletion isolation, constraint deletion scope, storage lifecycle
guards, operation completion/size filtering and departed-unit membership.
No schema/generated files, service/API behavior or transaction changes.
Local implementation commit: `8ce0410f515a9b46bc09ba006c108852267eb8e7`.
All eleven expressions use explicit single-column membership subqueries;
existing CTEs, joins, lifecycle guards and transaction behavior are preserved.
Added `TestGetConsumerRelationUnitsChangeDepartureScope` for empty/partial/full
membership and cross-relation/application isolation.
Rebased without conflicts onto refreshed `upstream/4.1`
`5259edff4329ca4a1de80d77e0d614b832d4ad72`; new local head
`02e9f73e4936d16f8acb23aaeae5e35228b315b7`. Range-diff confirms the patch is
unchanged. Published this head to the fork with an exact old-head lease;
verified the remote head. Worktree clean.

Validation with shared `_deps` and stock SQLite/Dqlite fixtures:
- Focused regressions across all four affected state packages passed with
  original and portable SQL (`make run-go-tests`, `-p 2 -race`).
- The new departure-scope regression also passed separately on original SQL.
- All four full state-package race suites passed: application, removal/model,
  operation and relation (`make run-go-tests`, `TEST_ARGS='-p 2 -race'`).
- Final relation package compiled with `-race -c`; bounded departure-scope
  stress completed 38 iterations with zero reported failures (two active
  when outer `timeout 31` stopped the run, exit 124). Binary removed.
- `make pre-check` passed (default skips static analysis); Go lint passed via
  `RUN_SUBTEST=run_go make static-analysis
  STATIC_ANALYSIS_JOB=test_static_analysis_go`.
- `gofmt`, prescribed `gci` and `git diff --check` passed. Inventory found no
  remaining uppercase SQL `IN table/cte` shorthand outside comments.
- After rebasing, all four affected state-package race suites passed again;
  `make juju` built a session-local client, and `juju version` ran successfully.
  The client binary was removed.

PR [juju/juju#23750](https://github.com/juju/juju/pull/23750) remains open
against `4.1`. On November 9, head `67daa565a6` included and answered
gfouillet's November 6 review requests, though the threads remained open.
SQLite race suites passed for the follow-ups; a real-dqlite run did not build
because `dqlite.h` was missing. The current head's revision smoke and
aggregate failed; human confirmation, real-dqlite validation and manual QA
remain. Earlier passing validation above is for `02e9f73e49`. The PoC retains
earlier equivalent SQL, not a verified reconciliation of the final upstream
result. No PostgreSQL execution or full repository suite was recorded.

### Historical CI evidence (2026-11-09)

The [revision smoke job](https://github.com/juju/juju/actions/runs/42653392922/job/129285455048)
failed after cleanup; its aggregate also failed. This snapshot does not
establish a shared cause with other PR failures or describe later heads.
