# Stage 08: PostgreSQL runtime integration

Branch `pg-poc/08-dbaccessor`, on `pg-poc/07-config`; current stack top.

## Status and prerequisites

Bootstrap/placement code is implemented; the runtime worker is not.
The latest handoff reports `TestPostgresBootstrapSuite` passing on PG.
The old operation-4 query failures were fixed on Stage 5; do not repeat
that work. Recover the tested revision/log or rerun before using this as
acceptance evidence ([Stage 6 results](06-compatibility.md#latest-recorded-evidence)).

Implementation resumes after the Stage 6 runtime-entry gate (D42), not
all broader compatibility work. Stage 7 supplies immutable config and
`connsource.Source`. Read D3, D4, D7, D9, D11, D19–D23 and N2 for runtime
work, plus X13/X22 for pool policy.

Already present: `postgresql.Placement` (`SchemaName`, `OpenAdmin`,
`OpenNamespace`), schema create/ensure/drop/existence helpers,
`WithRootCAs`, `database.DqliteInitialiser` / `PostgresInitialiser`,
`agentbootstrap.DatabaseInitialiser` and selection at
`cmd/jujuagentd/agent/bootstrap.go` (`newDatabaseInitialiser`).
Verify APIs on the current branch; these are not requests to rebuild them.

## 08-contract: settle the remaining lifecycle contract

Before assigning implementation, inspect `internal/worker/dbaccessor`
interfaces, `tracker.go`, the machine/safemode manifolds and
`internal/worker/upgradedatabase`. Record the existing shared semantics
and approve any new choices:

- Fresh bootstrap rejects existing controller/initial-model schemas (D23).
  Restart and HA joining must reuse existing schemas without invoking
  fresh-bootstrap rejection. New model namespaces initialise lazily.
- Define readiness: schema existence alone does not establish completed
  DDL. Specify the schema-qualified initialisation/version check and
  coordination of concurrent initialisers before publishing a usable DB.
- Define pool ownership, cancellation, shutdown/restart and error/retry
  boundaries. Preserve the shared interface contract; do not require
  callers to know concrete worker types or undocumented call ordering.
- Specify deletion completion, repeated deletion, outstanding users and
  acquire-versus-delete behaviour before implementing `DeleteDB`.
- Identify exactly which dqlite cluster hooks become no-ops. Juju controller
  membership, leases and watchers are still required; D19 removes the
  embedded database's node lifecycle, not controller HA.

Output: a short approved contract here, with remaining uncertainty marked.
Until then, the following are bounded task outlines, not fully dispatchable
implementation instructions.

## Implementation tasks

| Task | Prerequisites and scope | Required checks |
|---|---|---|
| 08-tracker | Approved contract; replace SQLite catalog/PRAGMA assumptions and select model DDL by backend. | Fresh/existing namespace, concurrent initialisation, DDL failure, cancellation, namespace isolation; default dqlite unchanged. |
| 08-worker | Tracker contract; separate `pgWorker` reusing `trackedDBWorker`, connection source and machine/safemode output wiring. | Backend/default selection, ready access, restart/cancellation and deterministic pool closure. |
| 08-upgrade-selector | Backend plumbing; wire `ControllerDDLForBackend` / `ModelDDLForBackend` into the upgrade worker. | Both selector paths and existing lifecycle semantics; no new restore/cross-version upgrade support. |
| 08-delete | Approved deletion contract and worker ownership. | Teardown reaches the intended schema; repeated calls, cancellation and acquire/delete races; unrelated schemas survive. |
| 08-cleanup-tool | Separate operator task; `scripts/postgresql-poc/drop-juju-schemas.py`. | Non-destructive inventory, explicit target/confirmation and controller-stopped precondition; never blanket cleanup on bootstrap failure. |

Pools must retain SERIALIZABLE, UTC and namespace search paths; preserve
the charm's login role and apply the cap of 3 connections per namespace.
D22 applies CA data in memory. Clarify removal of provisioning-host
`sslrootcert` paths from agent URLs before live TLS acceptance; do not
assume the path exists on controller machines.

## Acceptance

Each task needs focused unit/integration tests and the required race/stress
coverage. Bootstrap-only success does not accept runtime worker changes.
Stage 9 must then exercise real bootstrap, added-model creation, restart
and completed deletion on both PoC backends. Unexpected shared-SQL failures
return to their owning stage; record them rather than hiding them in this
worker commit. U11 must land before the functional milestone (D40).
