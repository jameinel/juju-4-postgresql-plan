# Stage 08: PostgreSQL runtime integration

Branch `pg-poc/08-dbaccessor`, on `pg-poc/07-config`; current stack top.

## Status and prerequisites

Bootstrap/placement code, 08-tracker, 08-worker, 08-upgrade-selector
and 08-delete are implemented; 08-cleanup-tool is not. The lifecycle contract below
was approved on 2026-10-26 (D44–D47); D48 records the tracker interfaces,
D49 the manifold's connection source, D50 the loss of the controller
schema and D51 the upgrade worker's backend.
`TestPostgresBootstrapSuite` passes on PG (2 tests) in the D42 run at
`08=c5f185e6f6`. The old operation-4 query failures were fixed on Stage 5;
do not repeat that work ([D42 evidence](06-compatibility.md#d42-runtime-entry-gate-run)).

Implementation resumes after the Stage 6 runtime-entry gate (D42), which
was met on 2026-10-26; broader compatibility work continues separately.
Stage 7 supplies immutable config and `connsource.Source`. Read D3, D4,
D7, D9, D11, D19–D23, D44–D47 and N2 for runtime work, plus X13/X22 for
pool policy.

Already present: `postgresql.Placement` (`SchemaName`, `OpenAdmin`,
`OpenNamespace`), schema create/ensure/drop/existence helpers,
`postgresql.SchemaInitialised` (the D44 marker check), `postgresql.DeleteSchema`
(the D47 drop), `WithRootCAs`,
`database.DqliteInitialiser` / `PostgresInitialiser`,
`agentbootstrap.DatabaseInitialiser` and selection at
`cmd/jujuagentd/agent/bootstrap.go` (`newDatabaseInitialiser`), and the
tracker's `DBOpener` / `TrackedDBBackend` interfaces with
`dbaccessor.PostgresNamespaces` (D48), and `dbaccessor.pgWorker` with
manifold selection from `DBConnectionSource` (D49), and the upgrade
worker's schema selection from the same source (D51), and
`PostgresNamespaces.Delete` behind `pgWorker.DeleteDB`.
Verify APIs on the current branch; these are not requests to rebuild them.

## 08-contract: approved lifecycle contract

Approved 2026-10-26. The shared semantics below are unchanged; this is how
the PostgreSQL worker maps them.

**Interface.** The manifold still exports `DBGetter`, `DBDeleter` and
`ClusterDescriber`. `GetDB` blocks until the worker is ready. It returns
`ErrDBAccessorDying` while the worker is dying, and `ErrDBNotFound` for a
namespace missing from `namespace_list`; the controller namespace is
always allowed. Requests stay serialised through the worker loop, with
one `trackedDBWorker` per namespace. `ErrDBDead` and `ErrDBNotFound` stay
non-fatal to the runner. Callers need no concrete worker type.

**Start, restart and HA join.** `pgWorker` reads the `connsource` details
once; they are fixed at bootstrap (D9). Before serving it opens the
controller tracker, checks the controller marker (D44) and calls
`AddDqliteNodeID` (D45). It never creates schemas or applies DDL for the
controller, and it never rejects existing state: D23's rejection belongs
to bootstrap alone. On any failure the worker returns an error and the
dependency engine restarts it.

**Model namespaces.** On the first `GetDB`, the tracker opens its pool and
runs `EnsureSchema` through it. It then checks the marker and applies
`ModelDDLForBackend(BackendPostgreSQL)` only when the marker is absent.
All of this happens before the tracker is published. D44 assumes a
single initialiser.

**Pools.** Each tracker owns one `Placement.OpenNamespace` pool, closed
when the tracker stops. The pool uses SERIALIZABLE, UTC, the namespace's
`search_path`, the D22 in-memory CA and the charm's login role, and is
capped at 3 connections. Transactions keep the existing tomb-bound
context and `database.Retry` policy. The ping/reopen loop stays; the
ping also checks that the schema still exists. A tracker whose model
schema is gone exits with `ErrDBDead` (D46, provisional); a missing
controller schema is a plain error that fails the worker (D50). There
is no long-lived admin pool.

**Delete.** `DeleteDB` keeps the existing order: refuse the controller
namespace, kill the local tracker with `ErrDBDead` and wait for it, then
drop the schema through a short-lived admin pool (D47). Deletion is
complete when the drop transaction commits. Deleting again, or deleting
a schema that was never created, succeeds. Local callers that still hold
the `TxnRunner` get `ErrDBDead`; trackers on other controllers exit at
their next ping (D46). Model removal deletes the `namespace_list` row
first, so a later `GetDB` fails the namespace check unless another
controller already has the tracker cached; that tracker keeps answering
until its ping fails.

**dqlite hooks that become no-ops.** The PostgreSQL worker has none of
these: `NodeManager`, the dqlite app or data directory, cluster join,
`handleClusterConfigChange` (controller agent config watching), or
handover on shutdown. `ClusterDescriber` returns no nodes (D19), so
status omits cluster roles and votes. `AddDqliteNodeID` replaces
`AddDqliteNode`. The `db-repl` command (`dbrepl`, `dbreplaccessor`) stays
dqlite-only. Controller membership (`controller_node`), leases,
change-stream watchers and API address publication are unchanged.

**Remaining uncertainty.**

- On HA, a `GetDB` on one controller can pass the `namespace_list`
  check while another controller deletes the model, and recreate an
  empty schema. dqlite has the same race. Record it if seen; no fix is
  planned.
- The 15 s `lock_timeout` on deletes is not tuned. It bounds each lock
  wait, not the whole delete, and the delete holds up uncached `GetDB`
  and `DeleteDB` requests while it waits ([08-delete result](#08-delete-result)).
- A cancelled PostgreSQL query leaves its server backend running until it
  completes or times out: pgx closes the connection without sending a
  cancel request. This applies to every PoC query; deletes are bounded by
  their lock timeout.
- Upgrades of PostgreSQL schemas are exercised only by re-applying the
  backend's DDL to an initialised schema (08-upgrade-selector). No
  PostgreSQL schema patch exists yet, and the data-migration upgrade
  steps are untested on PostgreSQL.

## Implementation tasks

| Task | Prerequisites and scope | Required checks |
|---|---|---|
| 08-tracker | **Done** ([result](#08-tracker-result)). Approved contract; replace the SQLite catalog/PRAGMA assumptions, select model DDL by backend and add the D46 schema check to the ping. | Fresh namespace, existing namespace and empty schema without a marker; a DDL failure leaves no marker; an overlapping initialiser fails cleanly and its retry succeeds; cancellation; namespace isolation; a dropped schema ends the tracker with `ErrDBDead`; default dqlite unchanged. |
| 08-worker | **Done** ([result](#08-worker-result)). Tracker contract; separate `pgWorker` reusing `trackedDBWorker`, connection source and machine/safemode output wiring. | Backend/default selection; a missing controller marker is an error; `AddDqliteNodeID` runs before ready; ready access; restart/cancellation; deterministic pool closure. |
| 08-upgrade-selector | **Done** ([result](#08-upgrade-selector-result)). Backend plumbing; wire `ControllerDDLForBackend` / `ModelDDLForBackend` into the upgrade worker. | Both selector paths and existing lifecycle semantics; no new restore/cross-version upgrade support. |
| 08-delete | **Done** ([result](#08-delete-result)). Approved contract and worker ownership; D47 RESTRICT drop replacing the interim `NotImplemented` in `pgWorker.deleteDatabase`. The CASCADE `postgresql.DropSchema` stays, for tests only. | Teardown reaches the intended schema; repeated and never-created deletes; an unexpected object or an outside dependent fails without dropping anything; lock timeout, cancellation and acquire/delete races; unrelated schemas survive. |
| 08-cleanup-tool | Separate operator task; `scripts/postgresql-poc/drop-juju-schemas.py`. | Non-destructive inventory, explicit target/confirmation and controller-stopped precondition; never blanket cleanup on bootstrap failure. |

Clarify removal of provisioning-host `sslrootcert` paths from agent URLs
before live TLS acceptance; do not assume the path exists on controller
machines.

## 08-tracker result

Done 2026-10-26 at `08=45f3bdf358` (pushed), in three commits:

- `b6a7690d80` `fix(dbaccessor): close databases the tracker opens but
  does not keep`: U14, three pool leaks that also exist on stock 4.1.
- `f9a8adfbf1` `feat(database): report whether a postgresql schema is
  initialised`: `postgresql.SchemaInitialised`, the D44 marker check.
  `TestSchemaInitialised` covers a missing schema, an empty schema, a view
  named `schema`, a marker in another schema, a marker with no rows and a
  complete marker.
- `45f3bdf358` `feat(dbaccessor): add tracked database backends for
  postgresql`: D48. `PostgresNamespaces` opens the namespace pool, runs
  `EnsureSchema` outside a transaction, checks the marker and applies
  `ModelDDLForBackend(BackendPostgreSQL)` only when it is absent. Its ping
  returns `ErrDBDead` when the schema is gone (D46). Controller trackers
  never create or initialise the schema; 08-worker must check the
  controller marker before serving.

Required checks, `TestTrackedDBPostgresSuite`
(`internal/worker/dbaccessor/tracker_pg_test.go`), all on PostgreSQL:

| Check | Test |
|---|---|
| Fresh namespace | `TestFreshNamespace`: schema created, marker present, 156 version rows, `current_schema()` is the namespace. |
| Existing namespace | `TestExistingNamespace`: with a tampered hash that makes re-applying the DDL fail, a new tracker starts and data survives. |
| Empty schema without a marker | `TestEmptySchemaWithoutMarker` |
| DDL failure leaves no marker | `TestDDLFailureLeavesNoMarker`: a conflicting `change_log` fails the DDL; no `schema` table remains; a retry after removing the conflict succeeds. |
| Overlapping initialisers | `TestOverlappingInitialisers`: 4 concurrent trackers; the losers fail with 23505 or 42P07 (DIALECT.md X39) and leave no marker; each retry succeeds. |
| Cancellation | `TestCancelledInitialisationLeavesNoMarker`: DDL blocked on an uncommitted conflicting table (confirmed through `pg_blocking_pids`), then cancelled; no marker; a retry succeeds. |
| Namespace isolation | `TestNamespaceIsolation` |
| Dropped schema | `TestDroppedSchemaEndsTrackerWithErrDBDead`: `Wait` returns `ErrDBDead`, `Open` ran once, the schema stays dropped, `StdTxn` returns `ErrDBDead`, and another namespace's tracker stays healthy. |
| Controller namespace | `TestControllerNamespaceIsNotCreated`: the schema is not created and the first ping ends the tracker. 08-worker changed the error from `ErrDBDead` to a plain error (D50). |
| Default dqlite unchanged | Existing `TestTrackedDBWorkerSuite` and the full package pass on SQLite; new `TestWorkerDoesNotReopenDeadDB`. |

Both `ErrDBDead` tests fail when the no-reopen short-circuit is disabled
(3 opens instead of 1). Validation: the full `dbaccessor` package with
`-race` on SQLite and on PostgreSQL, `internal/database/postgresql` on
PostgreSQL, `go build ./...`; stress with `-race`: tracker suite on SQLite
220 runs, PostgreSQL tracker suite 18 runs (`-p 3`), 0 failures.

## 08-worker result

Done 2026-10-26 at `08=aa09e41411` (pushed), in three commits:

- `5594111e4b` `refactor(dbaccessor): extract the tracked database
  runner`: `newTrackedDBRunner`, so both workers share the runner's
  fatal and restart rules.
- `fa46a683cc` `feat(dbaccessor): add a postgresql dbaccessor worker`:
  `pgWorker` (`internal/worker/dbaccessor/pgworker.go`). Before `GetDB`
  unblocks, it starts the controller tracker, checks the controller
  marker with `PostgresNamespaces.CheckControllerInitialised` (D44) and
  calls `AddDqliteNodeID` (D45). Request serialisation, the
  `namespace_list` check and the tracker cache follow the dqlite worker.
  The controller tracker's ping returns a plain error when the schema is
  gone (D50).
- `aa09e41411` `feat(dbaccessor): select the dbaccessor backend from
  agent config`: D49. The output `ClusterDescriber` reports no nodes
  (D19); `domain/status` builds an empty map from that.

**Interim delete (superseded by [08-delete](#08-delete-result)).**
`DeleteDB` refused the controller namespace with `Forbidden`, killed the
namespace's tracker with `ErrDBDead` and waited for it, then returned
`NotImplemented`; the schema survived, so model removal on PostgreSQL
could not complete.

**Race fixed while testing.** When the controller tracker fails to
start, the runner dies with the real error, but `start` can see
`worker.ErrDead` first. The loop then returns the runner's error instead.

Required checks:

| Check | Test |
|---|---|
| Backend/default selection | `TestStartPostgreSQL` (outputs, empty cluster details, no dqlite inputs), `TestStartDqliteNeedsDqliteInputs`, `TestStartUnknownBackend`, `TestStartConnectionDetailsError` and a nil-source `Validate` case in `manifold_test.go`; `TestDetailsReadCurrentAgentConfig` (empty backend is dqlite) and `TestSafeModeDetailsReadCurrentAgentConfig` in `cmd/jujuagentd/agent`. |
| Missing controller marker | `TestMissingControllerSchema` (no schema is created) and `TestControllerSchemaWithoutMarker`. |
| `AddDqliteNodeID` before ready | `TestRegistersControllerBeforeServing` (rows `0`, `1`); `TestControllerNodeNotAlive` (the worker fails and never serves). |
| Ready access | `TestGetDBModelNamespaces`: controller and model namespaces, the cached runner, a registered namespace without a schema gets an initialised one, and an unregistered namespace returns `ErrDBNotFound` without a schema. |
| Restart/cancellation | `TestRestart`; `TestKillWhileRegistering`: registration blocked on an `ACCESS EXCLUSIVE` lock (confirmed through `pg_blocking_pids`), a cancelled `GetDB` returns `context.Canceled`, `Kill` is clean and later calls return `ErrDBAccessorDying`. |
| Deterministic pool closure | `TestKillClosesPools`: sessions tagged with `application_name` drop to 0 after `Kill`. |
| Controller schema loss (D50) | `TestDroppedControllerSchemaIsFatal`, and the amended `TestControllerNamespaceIsNotCreated`. |
| Interim delete | `TestDeleteDB`: `Forbidden` for the controller, `NotImplemented` for a model, the old runner returns `ErrDBDead`, and the schema survives. Superseded: 08-delete changed it to expect the drop. |

Validation: the full `dbaccessor` package with `-race` on SQLite and on
PostgreSQL, the `cmd/jujuagentd/agent`, `machine` and `safemode`
packages, `go build ./...` and `golangci-lint` on the changed packages (0
issues); stress with `-race` of the PostgreSQL worker and manifold
suites, 24 runs (`-p 3`), 0 failures.

## 08-upgrade-selector result

Done 2026-10-27 at `08=306db7518f` (pushed), in one commit:
`306db7518f` `feat(upgradedatabase): select the upgrade schemas by
backend` (D51). The upgradedatabase manifold reads the backend from
`DBConnectionSource`; the machine agent passes `StartupValueProvider`, as
for dbaccessor. Safe mode does not run the upgrade worker.

Before this change the worker applied the dqlite DDL to PostgreSQL. On
PostgreSQL, `TestUpgradeController` aborted with a syntax error at
`AUTOINCREMENT`. The unexpected `SetDBUpgradeFailed` mock call then
stopped the worker goroutine, and the test's deferred kill waited for the
whole package timeout.

Required checks:

| Check | Test |
|---|---|
| Backend mapping | `TestStart` (dqlite), `TestStartPostgreSQL`, `TestStartUnknownBackend` (no worker starts), `TestStartConnectionSourceError` and a nil-source `Validate` case in `manifold_test.go`; `TestValidateBackend` rejects an empty or unknown worker backend. |
| Both selector paths | The worker suite uses `schematesting.BackendForTest()`, so each run selects that backend's DDL. `TestUpgradeControllerUsesConfiguredBackend` (full lifecycle: abort, `SetDBUpgradeFailed`, `applying controller schema`) and `TestUpgradeModelUsesConfiguredBackend` configure the other backend, whose DDL fails on the test database, so a hardcoded schema fails on at least one backend. |
| Existing lifecycle semantics | The existing worker tests pass on both backends. `TestUpgradeControllerThatIsAlreadyUpgraded` and `TestUpgradeModelsThatIsAlreadyUpgraded` now apply the backend's DDL first, so the worker's `Ensure` is a no-op on an initialised schema. |

Validation: the `upgradedatabase` packages with `-race` on SQLite and on
PostgreSQL (16 s), the `machine` package, `go build ./...` and
`golangci-lint` on the changed packages (0 issues); stress with `-race`
of the worker and manifold suites on SQLite, 138 runs (`-p 3`), 0
failures.

## 08-delete result

Done 2026-10-27 at `08=fc8f6f7f88` (pushed), in two commits:

- `7d9198a0f1` `feat(database): drop postgresql namespace schemas without
  cascade`: `postgresql.DeleteSchema(ctx, db, name, lockTimeout)` (D47).
  One transaction with `SET LOCAL lock_timeout` checks that the schema
  exists (a missing one succeeds), then drops its views, tables,
  functions and remaining sequences by name, then the schema, each
  statement with RESTRICT. Anything else (types, materialized views,
  procedures, aggregates) or an outside dependent fails with 2BP01 and
  nothing is dropped. PostgreSQL lists the dependents only in the error's
  `Detail`, so the error appends it (X26, X41). A lock timeout under 1 ms
  is rejected, because 0 disables it. `DropSchema` (CASCADE) is
  documented as test-only.
- `fc8f6f7f88` `feat(dbaccessor): delete postgresql model schemas`:
  `PostgresNamespaces.Delete` opens a short-lived admin pool and calls
  `DeleteSchema` with a 15 s lock timeout; `pgWorker.deleteDatabase` calls
  it after killing the tracker and waiting for it, replacing the interim
  `NotImplemented`.

**Isolation.** The delete transaction uses READ COMMITTED, an exception
to D7 approved on 2026-10-27 and recorded as a D47 amendment. Under a
SERIALIZABLE snapshot, objects dropped by a concurrent delete are still
listed, and `regclass` renders their OIDs as bare numbers; the `DROP`
then fails with `syntax error at or near "<oid>"`. SSI does not track
catalog reads (X40).

**Behaviour.**

- The delete runs on the worker loop, so uncached `GetDB` and `DeleteDB`
  requests wait behind it; cached runners are still served.
- `lock_timeout` bounds each lock wait, not the whole delete. 15 s is not
  tuned.
- A tracker still starting is waited for: schema creation and DDL run
  before the tracker is published, so a delete never overlaps the DDL.
- Cancelling or killing the worker mid-delete drops nothing. The caller
  gets `ErrDBAccessorDying` or `context.Canceled`, depending on a select
  race. pgx v5's default `DeadlineContextWatcherHandler` closes the
  connection without a cancel request, so the server backend keeps
  waiting until it gets the lock or times out, then rolls back. This
  affects every PoC query; `pgconn.CancelRequestContextWatcherHandler`
  is a possible follow-up (not verified).

**Test fixture finding.** `BootstrapPostgres` creates and initialises
the controller model's schema as well as the controller's. The first
delete tests used that model, so the in-progress-start case never ran.
The worker delete tests now register a fresh namespace, check that its
schema is absent, and `TestDeleteDB` checks that the controller model's
schema survives.

Required checks. `TestSchemaSuite` and `TestPlacementSuite` are in
`internal/database/postgresql`; `TestPgWorkerSuite` is in
`internal/worker/dbaccessor/pgworker_pg_test.go`. All but
`TestDeleteSchemaRejectsShortLockTimeout` and `TestWithDetail` run on
PostgreSQL.

| Check | Test |
|---|---|
| Teardown reaches the intended schema; unrelated schemas survive | `TestDeleteSchemaDropsJujuObjects`: tables, identity and serial columns, a sequence, an index, a trigger and its function, overloaded functions and nested views; neighbours `<name>x` and `other` with the same objects are unchanged. `TestDeleteDB`: the old runner returns `ErrDBDead`, the schema is gone, a later `GetDB` returns `ErrDBNotFound` without recreating it, and the controller, controller model and another model survive. |
| Repeated and never-created deletes | `TestDeleteSchemaDropsJujuObjects`, `TestDeleteSchemaEmpty`, `TestDeleteDBWithoutTrackedDatabase` (a schema created by an earlier worker, then deleted twice, and a namespace that never had one). |
| Unexpected object fails without dropping anything | `TestDeleteSchemaUnexpectedObject` (enum, materialized view, procedure, aggregate); `TestDeleteDBForeignObject`, whose error is not `NotFound`, which the undertaker would take as success. A retry after removing the object succeeds. |
| Outside dependent fails without dropping anything | `TestDeleteSchemaOutsideDependent` (view, foreign key, trigger, sequence default); `TestDeleteDBForeignObject` (view). Neither schema changes; a retry after removing the dependent succeeds. |
| Lock timeout | `TestDeleteSchemaLockTimeout` (55P03 in well under the session's own timeout; nothing dropped; a retry succeeds), `TestDeleteSchemaRejectsShortLockTimeout`. |
| Lock wait | `TestDeleteSchemaWaitsForLock`; `TestDeleteDBWaitsForLock` (blocking confirmed through `pg_blocking_pids`; a cached runner is served meanwhile). |
| Cancellation | `TestDeleteSchemaCancelled`; `TestKillWhileDeleting` (no relation dropped, and the worker's sessions reach 0 once the lock is released). |
| Acquire/delete races | `TestDeleteDBRacesUse` (4 users loop on `GetDB` and transactions until `ErrDBNotFound`); `TestDeleteDBWhileStarting` (a tracker held after its DDL: the delete does not finish within 100 ms, then succeeds after release, and the runner returns `ErrDBDead`); `TestDeleteDBFromAnotherController` (the other controller's runner returns `ErrDBDead` and is not replaced); `TestDeleteSchemaConcurrentlyDeleted`. |
| Error detail | `TestWithDetail`. |

Mutations, each reverted:

| Mutation | Failing tests |
|---|---|
| SERIALIZABLE instead of READ COMMITTED | `TestDeleteSchemaConcurrentlyDeleted` |
| A fixed lock timeout instead of the caller's | `TestDeleteSchemaLockTimeout` |
| CASCADE instead of RESTRICT | `TestDeleteSchemaOutsideDependent`, `TestDeleteSchemaUnexpectedObject` |
| No tracker kill before the drop | `TestDeleteDB`, `TestDeleteDBWhileStarting` |
| No wait for a starting tracker | `TestDeleteDBWhileStarting` |

Validation: `go build ./...`; the `dbaccessor` and `postgresql`
packages with `-race` on SQLite; the full `dbaccessor` package with
`-race` on PostgreSQL (45 s) and the full `postgresql` package on
PostgreSQL; `golangci-lint` on the changed packages, whose only findings
are two pre-existing `rangeint` hints in `driver_test.go`. Stress with
`-race` (`-p 1`), 0 failures: `TestPgWorkerSuite` 14 runs,
`TestSchemaSuite` 83 runs.

## Acceptance

Each task needs focused unit/integration tests and the required race/stress
coverage. Bootstrap-only success does not accept runtime worker changes.
Stage 9 must then exercise real bootstrap, added-model creation, restart
and completed deletion on both PoC backends. Unexpected shared-SQL failures
return to their owning stage; record them rather than hiding them in this
worker commit. U11 must land before the functional milestone (D40).
