# PostgreSQL PoC: Stage 6 worker W1, stream fixtures (2026-11-10)

## Objective

Make the PostgreSQL test setup work for the change stream packages, and
skip two dqlite-only worker packages. The 2026-11-10 re-sweep at
`09=a3818782aa` recorded 94 failed leaf tests in five packages:

| Package | Failed leaves | Cause |
| --- | ---: | --- |
| `internal/changestream/stream` | 36 | dqlite `schema.ControllerDDL()` in setup |
| `domain/ssh/state/model` | 32 | same, applied to a second runner |
| `internal/worker/changestreampruner` | 3 | same |
| `internal/worker/dbrepl` | 19 | approved skip (dqlite REPL) |
| `internal/worker/dbreplaccessor` | 4 | approved skip (dqlite pragma) |

Each of the first four fails in `SchemaApplier.Apply` with
`syntax error at or near "AUTOINCREMENT" (SQLSTATE 42601)`: the setup
applies the dqlite controller or model DDL to PostgreSQL. With setup
fixed, the stream tests' own assertions run on PostgreSQL for the first
time, so expect further failures.

Follow the
[Worker protocol](2026-11-10-stage-6-parallel-repairs.md#worker-protocol)
for workspace, scope, test commands, commits and the report.

## Starting state

- The integrator gives you the worktree (`../juju-4-pg-w1`), branch
  (`pg-work/w1-stream-fixtures`), base commit and `-p` budget.
- Evidence: [06-compatibility.md](../stages/06-compatibility.md#postgresql-re-sweep)
  and the sweep JSONL named in the
  [parallel-repairs handoff](2026-11-10-stage-6-parallel-repairs.md#starting-state).
- Precedent: `domain/schema/testing/controllersuite.go:35,50` already
  applies `schema.ControllerDDLForBackend(BackendForTest())`.

## Work and boundaries

1. **Backend DDL in setup** (`Classification: portability`, `Dialect`:
   DIALECT.md section 4, `AUTOINCREMENT`; `Upstream-Home: none`, since
   the backend switch in `domain/schema/postgresql.go` is PoC-only).
   Replace the dqlite DDL with
   `schema.ControllerDDLForBackend(schematesting.BackendForTest())`, and
   assert its error, at:
   - `internal/changestream/stream/package_test.go:33-38`
     (`baseSuite.SetUpTest`);
   - `domain/ssh/state/model/state_test.go:35-42` (the controller
     runner from `s.OpenDB`);
   - `internal/worker/changestreampruner/package_test.go:31-46`
     (`SetUpTest` and `ApplyDDLForRunner`).

   Search each package's other test files for `schema.ControllerDDL()`
   or `schema.ModelDDL()` and convert them too.
2. **Stream tests.** Run `internal/changestream/stream` on PostgreSQL and
   classify what fails. Known site: `stream_test.go:1978` reads
   `result.LastInsertId()`, which pgx does not support (X35). Use
   `INSERT ... RETURNING id` and scan the id, as D36 and the X35 fix did;
   keep the closure reset (`ids = ids[:0]`) for txncheck. Expect order
   (X17), timestamp precision (X27) and bind type (X23, X25) failures in
   change log fixtures. Keep the stream's data shape and assertions; an
   ordering assertion that matches the stream's `ORDER BY id` contract
   is not X17. Report production stream SQL that is not portable;
   do not change it.
3. **ssh and pruner tests.** Run both packages after step 1 and repair
   new test-only failures under the protocol's repair rules.
4. **Approved skips** (N21; `Classification: portability`,
   `Upstream-Home: none`). Use
   `databasetesting.SkipOnPostgreSQL(c, reason)` from
   `internal/database/testing`, called before the embedded `SetUpTest`
   so that no database is created:
   - `internal/worker/dbrepl`: `completerSuite.SetUpTest`
     (`completer_test.go:31`) and `dbReplSuite.SetUpTest`
     (`worker_test.go:37`). Reason: "the dbrepl worker queries
     sqlite_master" (X32; `completer.go:154`, `worker.go:388,466,583,596`).
     Leave `manifold_test.go` running.
   - `internal/worker/dbreplaccessor`: add a
     `trackedDBReplWorkerSuite.SetUpTest` in `tracker_test.go` that skips,
     then calls `s.dbBaseSuite.SetUpTest(c)`. Reason: "the tracked
     dbrepl worker sets SQLite pragmas" (`tracker.go:94`). Leave the
     manifold and worker suites running.

   Skip only these suites. The skip must not hide a failure that a
   fixture change could fix.

## Tests

- PostgreSQL and real dqlite: all five packages, `-p 2`, `-timeout 120s`
  inside `timeout 150s` (`timeout 300s` for the make target).
- Iterate on `internal/changestream/stream` with `-run` selectors.
- Run the backend-selection probe once for the stream suite and once
  for the ssh suite.
- On PostgreSQL the skipped suites must report skips, not passes; give
  the counts.

## Safety and stop conditions

- Do not change `schema.ControllerDDL()`, `ModelDDL()`, the generator,
  `domain/schema/testing` or the production stream code.
- Stop an item and report it when a failure points at production stream
  SQL, at the PostgreSQL controller schema, or at the shared harness.
- Stop the task and report if `ControllerDDLForBackend` fails on
  PostgreSQL for a reason other than these fixtures, or if the ssh suite
  cannot open a second PostgreSQL runner.
