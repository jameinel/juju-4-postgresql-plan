# Stage 08: PostgreSQL runtime integration

Branch `pg-poc/08-dbaccessor`, on `pg-poc/07-config`; current stack top.

## Status and prerequisites

Bootstrap/placement code is implemented; the runtime worker is not. The
lifecycle contract below was approved on 2026-10-26 (D44–D47).
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
`WithRootCAs`, `database.DqliteInitialiser` / `PostgresInitialiser`,
`agentbootstrap.DatabaseInitialiser` and selection at
`cmd/jujuagentd/agent/bootstrap.go` (`newDatabaseInitialiser`).
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
ping also checks that the schema still exists, and the tracker exits
with `ErrDBDead` when it does not (D46, provisional). There is no
long-lived admin pool.

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
- The 15 s `lock_timeout` on deletes is not tuned.
- Upgrades of PostgreSQL schemas are exercised only by selecting the DDL
  for the backend (08-upgrade-selector).

## Implementation tasks

| Task | Prerequisites and scope | Required checks |
|---|---|---|
| 08-tracker | Approved contract; replace the SQLite catalog/PRAGMA assumptions, select model DDL by backend and add the D46 schema check to the ping. | Fresh namespace, existing namespace and empty schema without a marker; a DDL failure leaves no marker; an overlapping initialiser fails cleanly and its retry succeeds; cancellation; namespace isolation; a dropped schema ends the tracker with `ErrDBDead`; default dqlite unchanged. |
| 08-worker | Tracker contract; separate `pgWorker` reusing `trackedDBWorker`, connection source and machine/safemode output wiring. | Backend/default selection; a missing controller marker is an error; `AddDqliteNodeID` runs before ready; ready access; restart/cancellation; deterministic pool closure. |
| 08-upgrade-selector | Backend plumbing; wire `ControllerDDLForBackend` / `ModelDDLForBackend` into the upgrade worker. | Both selector paths and existing lifecycle semantics; no new restore/cross-version upgrade support. |
| 08-delete | Approved contract and worker ownership; D47 RESTRICT drop replacing the CASCADE `postgresql.DropSchema`, which only tests use. | Teardown reaches the intended schema; repeated and never-created deletes; an unexpected object or an outside dependent fails without dropping anything; lock timeout, cancellation and acquire/delete races; unrelated schemas survive. |
| 08-cleanup-tool | Separate operator task; `scripts/postgresql-poc/drop-juju-schemas.py`. | Non-destructive inventory, explicit target/confirmation and controller-stopped precondition; never blanket cleanup on bootstrap failure. |

Clarify removal of provisioning-host `sslrootcert` paths from agent URLs
before live TLS acceptance; do not assume the path exists on controller
machines.

## Acceptance

Each task needs focused unit/integration tests and the required race/stress
coverage. Bootstrap-only success does not accept runtime worker changes.
Stage 9 must then exercise real bootstrap, added-model creation, restart
and completed deletion on both PoC backends. Unexpected shared-SQL failures
return to their owning stage; record them rather than hiding them in this
worker commit. U11 must land before the functional milestone (D40).
