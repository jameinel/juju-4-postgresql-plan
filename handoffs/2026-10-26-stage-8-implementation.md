# PostgreSQL PoC: handoff, Stage 8 implementation (2026-10-26)

> Snapshot for the next session. PLAN.md, the stage files and DECISIONS.md
> stay authoritative; where they disagree with this note, they win.

Goal: implement the PostgreSQL dbaccessor worker against the approved
[Stage 8 lifecycle contract](../stages/08-dbaccessor.md#08-contract-approved-lifecycle-contract)
(D44–D47), then return to the open Stage 5 and Stage 6 items. Follow the
PLAN.md session protocol:

- Ask the user before any design choice, one question at a time.
- Make one commit per concern on the owning stage branch, then restack
  and push.
- Never run two PostgreSQL test packages at once, and never print the
  database URL.
- Leave the user's untracked `PR-4.1-*.md` files in the plan repo alone.

## 1. Where things are

The code tree is clean on `pg-poc/08-dbaccessor`. The stack is linear and
every `pg-poc/*` branch is pushed.

| Branch | Tip |
|---|---|
| `pg-poc/05-queries` | `79f2e66049` |
| `pg-poc/06-test-harness` | `c52376b098` |
| `pg-poc/07-config` | `e0c68da786` |
| `pg-poc/08-dbaccessor` | `c5f185e6f6` |

Done since the [previous handoff](2026-10-25-stage-6-remaining.md):

- Stage 5: X37 (`0703122e2a`) and X38 (`18d679a2b7`), proposed upstream
  together as `4.1-sql-standard-syntax`. U11 (cross-model secret
  revisions, juju/juju#23688, still open) is cherry-picked unchanged as
  `8f2c49c355` and `79f2e66049` (D43).
- The D42 runtime-entry gate is met at `08=c5f185e6f6`: all 13 entry
  packages pass on SQLite and on PostgreSQL with `-race`, and the bad-URL
  proof fails as expected
  ([evidence](../stages/06-compatibility.md#d42-runtime-entry-gate-run)).
  `domain/schema` is accepted with a known skip gap (06-schema-skip).
- The `domain/status/state/model` failures (48) are all X27 time precision.
  The full `domain/removal/state/model` run has 47 failures, all
  test-side. Both are recorded in the Stage 6 task table.
- The Stage 8 lifecycle contract is approved: D44 single initialiser and
  marker readiness, D45 `AddDqliteNodeID`, D46 (provisional) schema-aware
  ping, D47 CASCADE-free delete. Plan commit `da1c26f`.

## 2. Next: Stage 8 implementation

Work in this order on `pg-poc/08-dbaccessor`; each task row in the stage
file lists its required checks. Every task needs tests on both backends;
the goroutine code needs `-race` and `stress`.

### 2.1 08-tracker (`internal/worker/dbaccessor/tracker.go`)

SQLite-specific code to replace or bypass on PostgreSQL:

- `openDatabase` (line 152) sets `PRAGMA foreign_keys` (line 170).
  PostgreSQL always enforces foreign keys.
- `ensureModelDBInitialised` (about line 177) queries `sqlite_master` for
  `change_log` and applies `schema.ModelDDL()`. On PostgreSQL:
  `EnsureSchema` through the namespace pool, the D44 marker check
  (`schema` table in `juju_<ns>` with at least one row, via `pg_catalog`)
  and `schema.ModelDDLForBackend(schema.BackendPostgreSQL)`.
- The ping (`defaultPingDBFunc`, settable with `WithPingDBFunc`) gains the
  D46 schema-existence check; when the schema is gone, the tracker exits
  with `database.ErrDBDead`.

The tracker only calls `DBApp.Open`, so it can be reused through a
narrower opener. `NewDBWorkerFunc` (`manifold.go:47`) takes a `DBApp`
today. How the tracker learns its backend, opener and DDL (a narrower
interface, new `TrackedDBWorkerOption`s, or something else) is a design
choice: propose it to the user before coding. Keep dqlite behaviour and
signatures used by existing callers unchanged where possible.

Pool helpers are in `internal/database/postgresql/placement.go`
(`OpenNamespace`, `OpenAdmin`, `EnsureSchema`, `SchemaExists`).
`applyDBLimits` already caps pools at 3 connections.

### 2.2 08-worker

A separate `pgWorker` in `internal/worker/dbaccessor` (D19), selected in
the manifold from `connsource` details (`internal/database/connsource`;
agent config `DBBackend` / `DBURL` / `DBCACert`). Reference points in the
dqlite worker (`worker.go`):

- `loop` 279, `GetDB` 429, `workerFromCache` 485, `DeleteDB` 547,
  `initialiseDqlite` 599 (the `AddDqliteNode` call at 625 becomes
  `AddDqliteNodeID`), `openDatabase` 699, `deleteDatabase` 740,
  `ensureNamespace` 1003.
- Runner `IsFatal` (about line 235): `ErrDBDead` and `ErrDBNotFound` are
  not fatal.
- No-ops on PostgreSQL: `NodeManager`, the dqlite app,
  `handleClusterConfigChange` 787, `rebindAddress` 885,
  `joinNodeToCluster` 925, `shutdownDqlite` 962.

`dbAccessorOutput` (`manifold.go:178`) type-asserts `*dbWorker`; it must
also accept `pgWorker` and export an empty `ClusterDescriber`. Wire the
selection into both manifolds: `cmd/jujuagentd/agent/machine/manifolds.go`
(about line 871) and `cmd/jujuagentd/agent/safemode/manifolds.go` (about
line 109).

### 2.3 08-upgrade-selector

`internal/worker/upgradedatabase/worker.go` calls `schema.ControllerDDL()`
(about line 505) and `schema.ModelDDL()` (about line 537). Select them
by backend.

### 2.4 08-delete

Implement D47 for `pgWorker.DeleteDB`. The existing
`postgresql.DropSchema` uses CASCADE; keep it for tests only. Callers are
`internal/worker/undertaker/worker.go` (about lines 260 and 342), which
retry and ignore NotFound.

### 2.5 08-cleanup-tool

Separate operator task: `scripts/postgresql-poc/drop-juju-schemas.py`
(see the stage file).

Then Stage 9 exercises real bootstrap, model creation, restart and
deletion on both backends.

## 3. Open items outside Stage 8

1. **05-SECRET-X16**: portable latest-revision queries in
   `domain/secret/state/state.go` (`updateSecret`, `GetRotationExpiryInfo`,
   `fetchSecrets`, `listCharmSecrets`,
   `getSecretsRevisionExpiryChanges`). Make it its own commit so it can be
   extracted as `4.1-secret-latest-revision`
   ([Stage 5](../stages/05-queries.md)).
2. **Stage 6 test repairs**
   ([task table](../stages/06-compatibility.md)): 06-schema-skip (make
   the `domain/schema` trigger and schema tests backend-aware),
   06-X27 (the 25 removal and 48 status failures), 06-X23, 06-X25,
   06-DDL, 06-X9, 06-X16, 06-trigger-drop, 06-errors and
   06-failure-injection.
3. **Possible harness speed-up**: only if the user asks. Per-test model
   DDL costs about 2.4 s because PostgreSQL fsyncs every index build
   ([OPERATIONS.md](../OPERATIONS.md#why-postgresql-state-tests-are-slow)).

## 4. Tooling and gotchas

- PostgreSQL runs: the `runpg.sh <logdir> <pkgs...>` helper in
  `~/.copilot/session-state/e1b8ccab-eefe-4bb9-a26f-d0c9097f66da/files/`
  (`GO_FLAGS`, `PKG_TIMEOUT`, `GO_TIMEOUT`). For manual runs, export
  `JUJU_POSTGRESQL_TEST_URL` from `~/.config/juju-pg-poc/db-url`, use
  `-count=1`, and pipe output through `cat` or `tee`; a direct `> file`
  redirect produced empty logs here.
- SQLite runs: `env -u JUJU_POSTGRESQL_TEST_URL`. Untagged runs use mattn
  SQLite. Tests of the dqlite node manager need
  `-tags dqlite,libsqlite3`.
- Suite isolation clears environment variables before `SetUpTest`. Read
  any probe flag at package init.
- Bad-URL proof: `postgres://nobody@127.0.0.1:1/none?connect_timeout=2`.
- The full `domain/removal/state/model` package takes about 22 minutes
  on PostgreSQL; `domain/status/state/model` takes about 8.
- Run `gci write --section standard --section default --section
  "Prefix(github.com/juju/juju)"` on every edited Go file.
- Restack and push:

  ```
  git switch pg-poc/08-dbaccessor
  git rebase --update-refs pg-poc/05-queries   # or the lowest changed stage
  git push --atomic --force-with-lease origin \
    $(git for-each-ref --format='%(refname:short)' 'refs/heads/pg-poc/*')
  ```

  Check `git merge-base --is-ancestor` for each adjacent pair and run
  `go build ./...` before pushing.
