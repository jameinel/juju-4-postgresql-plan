# PostgreSQL PoC: handoff, Stage 6 stream fixture (2026-11-10)

## Objective

Repair the PostgreSQL test setup for `internal/changestream/stream`. The
2026-11-10 re-sweep at code revision `09=a3818782aa` recorded 36 failed
leaf tests in this package; each surfaced PostgreSQL SQLSTATE 42601 at
SQLite-only `AUTOINCREMENT` DDL. Keep this task to the fixture path that
blocks this package. Do not repair the rest of the sweep or change
production queries as part of this task.

## Starting state

- Code repository: `jameinel/juju`, re-sweep revision
  `09=a3818782aa`, originally run on `pg-poc/09-bootstrap`.
- Stage 6 owns test harness and fixture repairs on
  `pg-poc/06-test-harness`; follow the current stack procedure in
  `PLAN.md` before switching branches or restacking.
- Plan repository: `main`, sweep-results commit `5f7a414`; this handoff
  is linked from [Stage 6](../stages/06-test-harness.md).
- The code worktree was clean after the sweep. Recheck both worktrees and
  branch tips before acting; preserve concurrent changes.
- The structured run is in
  `~/.copilot/session-state/5a9652bd-930f-45c8-a317-a4c57e24b91c/files/`
  `pg-sweep-a3818782aa.jsonl`. The current failure inventory and run
  details are in [Stage 6 compatibility evidence](../stages/06-compatibility.md#postgresql-re-sweep).

## Work and boundaries

1. Trace the `AUTOINCREMENT` DDL emitted during
   `internal/changestream/stream` test setup to its source. Check whether
   an existing dialect-aware test schema helper can express the intended
   generated identifier for both backends.
2. Make the smallest test-only, backend-compatible fixture change. Preserve
   the stream tests' data shape and assertions; do not alter production
   schema, stream SQL, or add skips. Classify and record the dialect
   difference under 06-dialect-fixtures and in `DIALECT.md`.
3. If the DDL comes from a shared helper and changing it would materially
   affect other packages, first inspect those callers and bound the change
   to demonstrated consumers. Stop and ask if the change requires a schema
   or production semantic decision.
4. Run the full `internal/changestream/stream` package on SQLite and
   PostgreSQL with `-race`. Confirm the package no longer fails during
   fixture setup and its stream behavior assertions execute. Record
   selectors, skips, logs, backend proof and revision. A fixed issue stays
   awaiting rerun until both backends pass.
5. Commit the fixture repair separately on the Stage 6 branch and update
   the current Stage 6 evidence. Do not push code branches without explicit
   approval.

## Safety and stop conditions

Use [OPERATIONS.md](../OPERATIONS.md) for PostgreSQL profile, environment,
progress monitoring and bounded runs. Do not overlap PostgreSQL test
invocations. Before testing, verify the target database and check for other
sessions or test schemas; do not drop or recreate schemas as part of this
fixture task. Keep credentials out of arguments and logs, and write logs
outside both repositories. Check long runs at least every two minutes.

Stop if the fixture fails for a reason other than the documented DDL,
production behavior appears implicated, the helper's wider impact is
unclear, or the test requires weakening assertions or introducing a skip.
The other failures in the 146-package sweep, including unitstate endpoint
ordering, grouped-column assertions, bind mismatches and changestream
worker failures, remain separate tasks.
