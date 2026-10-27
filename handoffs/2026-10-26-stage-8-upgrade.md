# PostgreSQL PoC: handoff, Stage 8 upgrade selector and delete (2026-10-26)

> Snapshot for the next session. PLAN.md, the stage files and DECISIONS.md
> stay authoritative; where they disagree with this note, they win.

Goal: finish Stage 8 with 08-upgrade-selector, then 08-delete, against the
approved [Stage 8 lifecycle contract](../stages/08-dbaccessor.md#08-contract-approved-lifecycle-contract).
Follow the PLAN.md session protocol:

- Ask the user before any design choice, one question at a time.
- Make one commit per concern on the owning stage branch, then push.
- Never run two PostgreSQL test packages at once, and never print the
  database URL.
- Leave the user's untracked `PR-4.1-*.md` files in the plan repo alone.

## 1. Where things are

The code tree is clean on `pg-poc/08-dbaccessor` at `aa09e41411`. Every
`pg-poc/*` branch is pushed; 08 is the top of the stack, so 08 commits
need no restack.

08-worker is done ([result](../stages/08-dbaccessor.md#08-worker-result)):

- `pgWorker` (`internal/worker/dbaccessor/pgworker.go`) serves tracked
  databases from PostgreSQL schemas. It registers the controller with
  `AddDqliteNodeID` before serving (D45). A lost controller schema is
  fatal (D50).
- The dbaccessor manifold reads `ManifoldConfig.DBConnectionSource` and
  selects the worker (D49). The machine and safe mode agents pass their
  startup value providers, which read `connsource` details from the
  current agent config.
- `pgWorker.deleteDatabase` refuses the controller, stops the tracker,
  then returns `NotImplemented`. Model removal on PostgreSQL does not
  complete until 08-delete.

Still open from earlier sessions: U14 has no UPSTREAM-PLAN.md task, and
D46 is still provisional.

## 2. Next: 08-upgrade-selector

`internal/worker/upgradedatabase/worker.go` calls `schema.ControllerDDL()`
in `upgradeController` (line 505) and `schema.ModelDDL()` in
`upgradeModel` (line 537). Both are dqlite. Select them with
`schema.ControllerDDLForBackend` / `ModelDDLForBackend`
(`domain/schema/postgresql.go`), which take a `schema.Backend`
(`BackendDqlite`, `BackendPostgreSQL`) and return an error for an unknown
backend.

Design questions to ask the user first:

- Where the upgrade worker learns the backend. The worker `Config`
  already has `Agent agent.Agent`, so it could read
  `connsource.NewAgentConfigSource(Agent.CurrentConfig())`. Alternatives
  are a `DBConnectionSource` on the upgrade manifold, mirroring D49, or a
  `schema.Backend` value resolved once in the manifold.
- Where to map `controller.DBBackend*` strings to `schema.Backend`. No
  shared helper exists; bootstrap has its own selection in
  `cmd/jujuagentd/agent/bootstrap.go` (`newDatabaseInitialiser`).

Required checks (stage file): both selector paths and the existing
lifecycle semantics; no new restore or cross-version upgrade support.

## 3. Then: 08-delete

Replace the interim `NotImplemented` in `pgWorker.deleteDatabase` with the
D47 drop: one transaction with `SET LOCAL lock_timeout` (15 s), objects
listed from `pg_catalog`, then views, tables, functions (by
`regprocedure`), remaining sequences and the schema, all with RESTRICT,
through a short-lived admin pool (`Placement.OpenAdmin`). Deleting a
missing schema succeeds. Keep the CASCADE `postgresql.DropSchema` for
tests only. Callers are `internal/worker/undertaker/worker.go` (lines 260
and 342), which retry and ignore NotFound. `TestDeleteDB` in
`pgworker_pg_test.go` asserts the interim behaviour and must change.

Required checks are in the stage file's task table.

## 4. Test commands

```
cd ~/dev/go/src/github.com/juju/juju-4-postgresql
go test -race -count=1 ./internal/worker/dbaccessor/
export JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url) && go test -race -count=1 ./internal/worker/dbaccessor/
go test -c -race -o /tmp/dbaccessor.test ./internal/worker/dbaccessor
export JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url) && timeout 200 ~/.local/bin/stress -p 3 -timeout 100s /tmp/dbaccessor.test -test.run '<Suite>'
```

`JUJU_POSTGRESQL_TEST_URL` is not set in a fresh shell, so a plain
`go test` runs on SQLite. Do not use `env -u`; the user's permission
prompt rejects it. Export the URL in the same command for PostgreSQL
runs. The dqlite worker suites are skipped when the URL is set
(`SkipOnPostgreSQL`), so run the package both ways. Run `gci` on every
edited Go file.

A PostgreSQL test that writes to a controller or model table with
triggers must use a pool opened with `Placement.OpenNamespace`: the
trigger functions write to an unqualified `change_log`, which the admin
pool's `search_path` does not reach.
