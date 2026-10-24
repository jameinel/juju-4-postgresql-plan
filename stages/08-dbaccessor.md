# Stage 08: PG dbaccessor worker and bootstrap

Branch `pg-poc/08-dbaccessor`, on `pg-poc/07-config`. Currently the top of
the stack.

- Goal: controller agents open namespaces on PostgreSQL.
- Steps:
  1. A PG `DBApp`/worker selected in the dbaccessor manifold when
     `db-backend=postgresql`. It needs no node lifecycle or handover.
  2. `ensureNamespace`: `CREATE SCHEMA IF NOT EXISTS`, then a pool with
     `search_path`; the tracker applies model DDL lazily (replace the
     `sqlite_master` check and the `PRAGMA`s). Schema names must not
     collide with the existing `public` and `login_hook` schemas (D20).
  3. `DeleteDB`: `DROP SCHEMA ... CASCADE`.
  4. `BootstrapPostgres` beside `BootstrapDqlite`; `controller_node` rows
     carry no dqlite IDs.
  5. Leave `ClusterDescriber` and the HA hooks as no-ops for PG.
- Verification: a bootstrap-only integration test against PG; the
  controller namespace and the first model namespace are created.
- Decisions: D19 to D23.
- Status: in progress. The bootstrap layer is committed (two commits); the
  worker is not started. Stage 6 was reopened during this stage, because
  its PostgreSQL suites had silently run on SQLite; which of the two goes
  first is open (see the PLAN.md status).
- Restart notes:
  - Done: `postgresql.Option` returns an error; `WithRootCAs`;
    `postgresql.Placement` (`SchemaName`, `OpenAdmin`, `OpenNamespace`)
    and `CreateSchema`/`EnsureSchema`/`DropSchema`/`SchemaExists`;
    `database.DqliteInitialiser` and `PostgresInitialiser`
    (`BootstrapPostgres`); `agentbootstrap.DatabaseInitialiser`;
    selection in `cmd/jujuagentd/agent/bootstrap.go`
    (`newDatabaseInitialiser`). Unit tests pass on SQLite; the database
    and placement tests pass on PostgreSQL.
  - Failing: the opt-in `TestPostgresBootstrapSuite`
    (`agent/agentbootstrap/fresh_state_postgresql_test.go`) stops at
    bootstrap operation 4. `ON CONFLICT ... DO UPDATE` seed queries use
    unqualified columns, which PostgreSQL rejects as ambiguous:
    `domain/access/state/user.go` (~1061),
    `domain/modeldefaults/state/state.go` (~510, ~599),
    `domain/modelconfig/state/state.go` (~291),
    `domain/relation/state/remoterelation.go` (~181, ~530),
    `domain/status/state/model/modelstate.go` (~589) and
    `domain/sequence/state/sequence.go` (~30, ~96, which also needs an
    `ON CONFLICT (namespace)` target). Qualify them with the table name
    (portability). These are shared queries, so the fix probably belongs
    on Stage 5 (`pg-poc/05-queries`); confirm with the user, then follow
    the PLAN.md stack workflow.
  - Next: the tracker `WithBackend` option (skip the foreign keys pragma,
    detect an initialised model through `information_schema`, use
    `ModelDDLForBackend`); `pgWorker` and manifold selection
    (`ControllerStartupValues.DBConnection`, `NewPGWorker` in the machine
    and safemode manifolds, `dbAccessorOutput`); `upgradedatabase` DDL
    selection by backend; `scripts/postgresql-poc/drop-juju-schemas.py`.
