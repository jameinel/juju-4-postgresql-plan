# PostgreSQL PoC: handoff, Stage 8 worker (2026-10-26)

> Snapshot for the next session. PLAN.md, the stage files and DECISIONS.md
> stay authoritative; where they disagree with this note, they win.

Goal: implement 08-worker, the PostgreSQL dbaccessor worker, against the
approved [Stage 8 lifecycle contract](../stages/08-dbaccessor.md#08-contract-approved-lifecycle-contract)
(D44–D48). Then do 08-upgrade-selector and 08-delete. Follow the PLAN.md
session protocol:

- Ask the user before any design choice, one question at a time.
- Make one commit per concern on the owning stage branch, then restack
  and push.
- Never run two PostgreSQL test packages at once, and never print the
  database URL.
- Leave the user's untracked `PR-4.1-*.md` files in the plan repo alone.

## 1. Where things are

The code tree is clean on `pg-poc/08-dbaccessor` at `45f3bdf358`. Every
`pg-poc/*` branch is pushed; 08 is the top of the stack, so 08 commits
need no restack.

08-tracker is done ([result](../stages/08-dbaccessor.md#08-tracker-result)):

- `b6a7690d80` U14: the tracker closed no pool it opened but did not
  keep. This is also a stock 4.1 bug and the commit applies unchanged to
  `upstream/4.1`. It is not proposed upstream yet. An UPSTREAM-PLAN.md
  task has not been written.
- `f9a8adfbf1` `postgresql.SchemaInitialised(ctx, Querier, schemaName)`,
  the D44 marker check.
- `45f3bdf358` D48: `NewTrackedDBWorker(ctx, opener DBOpener, namespace,
  opts...)`; `WithBackend(TrackedDBBackend)`; the dqlite backend is the
  default. `dbaccessor.NewPostgresNamespaces(placement)` implements both
  interfaces. A ping `ErrDBDead` ends the tracker without reopening.
  `NewDBWorkerFunc` takes a `DBOpener`.

## 2. Next: 08-worker

A separate `pgWorker` in `internal/worker/dbaccessor` (D19) that reuses
`trackedDBWorker` and has no node lifecycle, HA or handover logic.

### 2.1 Behaviour the contract requires

- Read the `connsource` details once (D9) and build one
  `postgresql.Placement{URL, CACert}` and one `PostgresNamespaces`.
- Start trackers through the runner exactly as the dqlite
  `openDatabase` does (`worker.go:699`, `NewDBWorker` call at 720), but
  pass `ns` as the opener and add `WithBackend(ns)`.
- Before serving (`close(dbReady)`): open the controller tracker, check
  the controller marker with `postgresql.SchemaInitialised` on
  `placement.SchemaName(ControllerNS)` (a missing marker is an error;
  never create the controller schema or apply its DDL), then call
  `nodeService().AddDqliteNodeID(ctx, controllerID)` (D45). This
  replaces `AddDqliteNode` (`worker.go:625`). On any failure return an
  error so the dependency engine restarts the worker.
- `GetDB`, `workerFromCache`, `ensureNamespace` and the request loop keep
  the dqlite semantics (`worker.go` 279, 429, 485, 1003).
  `ErrDBAccessorDying`, `ErrDBNotFound` and the runner `IsFatal` /
  `ShouldRestart` rules (`worker.go:235`) are unchanged.
- Deterministic pool closure: each tracker closes its pool when it stops
  (already true). Check that stopping the `pgWorker` stops every tracker
  and leaves no PostgreSQL backend sessions; `pg_stat_activity` filtered
  on the test's `application_name` or schema is one way.
- `DeleteDB` stays the dqlite order until 08-delete: refuse the
  controller namespace, kill the tracker with `ErrDBDead` and wait. The
  schema drop (D47) is 08-delete; until then do not drop anything, or
  return a clear "not implemented" error. Ask the user which.

### 2.2 Wiring

- `dbAccessorOutput` (`manifold.go:178`) type-asserts `*dbWorker`. It
  must also accept the `pgWorker` and export a `ClusterDescriber` that
  returns no nodes.
- `Manifold` (`manifold.go:101`) builds the dqlite `WorkerConfig`. It
  needs the backend from `connsource`; today `ControllerStartupValues`
  (`manifold.go:28`) has no database fields. Providers:
  `machineControllerStartupValueProvider` (`cmd/jujuagentd/agent/machine.go:98`)
  and `safeModeControllerStartupValuesProvider`
  (`cmd/jujuagentd/agent/safemode.go:342`). Both re-read
  `agent.CurrentConfig()`, which implements `connsource.Config`.
- Manifold call sites: `cmd/jujuagentd/agent/machine/manifolds.go`
  (about line 871) and `cmd/jujuagentd/agent/safemode/manifolds.go`
  (about line 109).
- The dqlite path must still need the controller agent config watcher;
  the PostgreSQL path has no use for it.

### 2.3 Design questions to ask the user first

1. How the manifold learns the backend: add database fields to
   `ControllerStartupValues`, or add a `connsource.Source` (or a factory)
   to `ManifoldConfig`. The bootstrap command already selects with
   `connsource.NewAgentConfigSource(cfg).Details(ctx)`
   (`cmd/jujuagentd/agent/bootstrap.go:390`).
2. Where the controller marker check lives: in the `pgWorker` start-up,
   or as a `PostgresNamespaces` method used by the worker.
3. What a controller tracker ending with `ErrDBDead` means. The runner
   treats `ErrDBDead` as not fatal and does not restart, which suits
   model namespaces. For the controller, a dropped controller schema
   would leave the worker serving with no controller tracker until the
   next `GetDB` starts one, which then dies at its first ping. Making it
   fatal to the `pgWorker` is one option.
4. Whether `DeleteDB` before 08-delete drops nothing or returns an error.

### 2.4 Required checks

From the stage table: backend/default selection; a missing controller
marker is an error; `AddDqliteNodeID` runs before ready; ready access;
restart and cancellation; deterministic pool closure. Use a real
PostgreSQL schema prefix (`pgtesting.NewSchemaPrefix`) and set the
controller schema up with `database.PostgresInitialiser` or the
bootstrap helpers so the marker and `controller_node` table exist. The
manifold output test should cover both worker types. Run `-race` and
`stress` (`-p` 2–4 for PostgreSQL).

## 3. After 08-worker

- 08-upgrade-selector: `internal/worker/upgradedatabase/worker.go`
  calls `schema.ControllerDDL()` (line 505) and `schema.ModelDDL()`
  (line 537). Select them by backend.
- 08-delete: D47 RESTRICT drop for `pgWorker.DeleteDB`. The existing
  `postgresql.DropSchema` uses CASCADE; keep it for tests only. Callers
  are `internal/worker/undertaker/worker.go` (lines 260 and 342), which
  retry and ignore NotFound.
- 08-cleanup-tool: separate operator task (see the stage file).

## 4. Test commands

```
export JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url)
go test -race -count=1 ./internal/worker/dbaccessor/
env -u JUJU_POSTGRESQL_TEST_URL go test -race -count=1 ./internal/worker/dbaccessor/
go test -c -race -o /tmp/dbaccessor.test ./internal/worker/dbaccessor
timeout 130 ~/.local/bin/stress -p 3 -timeout 100s /tmp/dbaccessor.test -test.run '<Suite>'
```

The dqlite worker suites are skipped when `JUJU_POSTGRESQL_TEST_URL` is
set (`SkipOnPostgreSQL`), so run the package both ways. Run `gci` on
every edited Go file.
