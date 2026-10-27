# PostgreSQL PoC: handoff, Stage 8 delete (2026-10-27)

> Snapshot for the next session. PLAN.md, the stage files and DECISIONS.md
> stay authoritative; where they disagree with this note, they win.

Goal: finish Stage 8 with 08-delete against the approved
[Stage 8 lifecycle contract](../stages/08-dbaccessor.md#08-contract-approved-lifecycle-contract).
Follow the PLAN.md session protocol:

- Ask the user before any design choice, one question at a time.
- Make one commit per concern on the owning stage branch, then push.
- Never run two PostgreSQL test packages at once, and never print the
  database URL.
- Leave the user's untracked `PR-4.1-*.md` files in the plan repo alone.

## 1. Where things are

The code tree is clean on `pg-poc/08-dbaccessor` at `c77f1ad694`. Every
`pg-poc/*` branch is pushed; 08 is the top of the stack, so 08 commits
need no restack. The stack was restacked on 2026-10-27 for the
PostgreSQL test session timeouts (N14, `06=4615e24467`), so earlier
08 hashes in the stage file are historical.

08-upgrade-selector is done
([result](../stages/08-dbaccessor.md#08-upgrade-selector-result)). The
upgradedatabase manifold reads the backend from the same
`DBConnectionSource` as dbaccessor, maps it to `schema.Backend`, and the
worker applies `ControllerDDLForBackend` / `ModelDDLForBackend` (D51).

08-worker is done ([result](../stages/08-dbaccessor.md#08-worker-result)):
`pgWorker` (`internal/worker/dbaccessor/pgworker.go`), manifold selection
(D49) and a fatal loss of the controller schema (D50).

Still open from earlier sessions: U14 has no UPSTREAM-PLAN.md task, and
D46 is still provisional.

## 2. Next: 08-delete

`pgWorker.deleteDatabase` (`pgworker.go`, line 294) refuses the controller
namespace with `Forbidden`, kills the namespace's tracker with
`ErrDBDead` and waits for it, then returns `NotImplemented` (line 316).
Replace that error with the D47 drop:

- One transaction with `SET LOCAL lock_timeout` (15 s), through a
  short-lived admin pool (`Placement.OpenAdmin`, `placement.go` line 59).
- List the schema's objects from `pg_catalog`, then drop views, tables,
  functions (by `regprocedure`), remaining sequences and the schema, all
  with RESTRICT.
- Deleting a missing schema succeeds.
- Keep the CASCADE `postgresql.DropSchema` (`placement.go` line 109) for
  tests only.

Callers are `internal/worker/undertaker/worker.go` (lines 260 and 342),
which retry with backoff and ignore NotFound. `TestDeleteDB`
(`pgworker_pg_test.go`, line 247) asserts the interim behaviour and must
change.

Required checks (stage file task table): teardown reaches the intended
schema; repeated and never-created deletes; an unexpected object or an
outside dependent fails without dropping anything; lock timeout,
cancellation and acquire/delete races; unrelated schemas survive.

After 08-delete, 08-cleanup-tool is a separate operator task, and the
stage file's acceptance section hands over to Stage 9.

## 3. Test commands

```
cd ~/dev/go/src/github.com/juju/juju-4-postgresql
set -o pipefail
timeout 150 go test -race -count=1 -timeout 120s -v ./internal/worker/dbaccessor/ 2>&1 | tee /tmp/dbaccessor-sqlite-$$.log | grep -E -- '^(FAIL|ok)|--- FAIL'
export JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url) && timeout 150 go test -race -count=1 -timeout 120s -v ./internal/worker/dbaccessor/ 2>&1 | tee /tmp/dbaccessor-pg-$$.log | grep -E -- '^(FAIL|ok)|--- FAIL'
go test -c -race -o /tmp/dbaccessor.test ./internal/worker/dbaccessor
export JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url) && timeout 120 ~/.local/bin/stress -p 1 -timeout 100s /tmp/dbaccessor.test -test.run '<Suite>'
```

- Bound every run and check progress at least every two minutes
  ([OPERATIONS.md](../OPERATIONS.md#bounded-runs-and-check-ins), N13).
  Pass `-v` so the log shows progress, and use `tee`: in this
  environment a plain `go test ... > file 2>&1` redirect has left the
  file empty. Use one stress process for PostgreSQL.
- PostgreSQL test sessions time out lock waits after 30 s (N14). D47's
  own `SET LOCAL lock_timeout` (15 s) overrides that inside the drop
  transaction, so a lock-timeout test sees D47's value.
- `JUJU_POSTGRESQL_TEST_URL` is not set in a fresh shell, so a plain
  `go test` runs on SQLite. Do not use `env -u`; the user's permission
  prompt rejects it. Export the URL in the same command for PostgreSQL
  runs. The dqlite worker suites are skipped when the URL is set
  (`SkipOnPostgreSQL`), so run the package both ways.
- Run `gci` on every edited Go file.
- A PostgreSQL test that writes to a controller or model table with
  triggers must use a pool opened with `Placement.OpenNamespace`: the
  trigger functions write to an unqualified `change_log`, which the admin
  pool's `search_path` does not reach.
