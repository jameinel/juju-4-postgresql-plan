# Stage 06: compatibility harness and acceptance

Branch `pg-poc/06-test-harness`, on `pg-poc/05-queries`.

## Status and purpose

The harness is implemented; the leaf-dialect refactor resolved its schema
import cycle. Remaining work is test repair, diagnosis and verification,
not building another harness. PostgreSQL bootstrap is reported passing;
this does not establish that all suites or the runtime worker work.

The latest recorded selection at `09=f3340c6710` passed on PostgreSQL and
real dqlite: 140 packages passed and two had no tests on each backend.
Skips and deferred packages remain coverage gaps; see
[the sweep evidence](06-compatibility.md#catalog-teardown-dependency-check-2026-11-11).
Stage 9 run 5 subsequently bootstrapped PostgreSQL, but full functional
acceptance remains open; see [Stage 9](09-bootstrap.md).

Stage 6 discovers compatibility defects; fixes can belong on Stages 3–5.
It supports bounded milestone gates rather than becoming a requirement to
make every Juju package pass before any runtime work resumes.

## Required context

Read the selected task and relevant rows in
[06-compatibility.md](06-compatibility.md), its X*/D* references, and
[OPERATIONS.md](../OPERATIONS.md) for running tests. Read N6, N9, N10,
N14, N15, N16 and D11 if changing the harness. Commit inventories and past
failures are in [history](../history/compatibility.md), not mandatory
setup reading.

## Existing contracts

- `internal/database/testing` and `domain/schema/testing` select PG via
  `JUJU_POSTGRESQL_TEST_URL`; absent the variable, existing SQLite/dqlite
  behaviour remains. Prove selection after environment/harness changes.
- Each test namespace has a unique throwaway schema (N9), with the
  matching controller/model DDL, deterministic closure and cleanup.
- `SchemaDialect()` returns the leaf `core/database/dialect` interface.
  Do not import the schema engine back into the harness and recreate the
  cycle. Use dialect-aware helpers for unavoidable backend-specific DDL.
- Follow up with an explicit test-suite backend identity API that can
  distinguish the SQLite fallback, real dqlite and PostgreSQL. The SQL
  dialect alone cannot distinguish the two SQLite-compatible backends.
  Until then, tests needing SQLite-only setup must combine the suite
  dialect with `dqlite.Enabled`; do not infer the backend from the build
  flag alone.
- PG connections use UTC, SERIALIZABLE and bounded pools (cap 3 per
  namespace, X22). Stage 8 must preserve these runtime requirements.
- PG test sessions get `lock_timeout` 30 s, `statement_timeout` 60 s and
  `idle_in_transaction_session_timeout` 60 s from `pgtesting.URL()`
  unless the URL sets them (N14, `06=4615e24467`). Passed on PostgreSQL
  with `-race`: the harness, driver, schema and blockcommand state
  packages at `06=4615e24467`; the harness, database, agentbootstrap,
  dbaccessor and upgradedatabase packages at the restacked top
  `08=c77f1ad694`.
- The per-test schema teardown fails the test if its CASCADE drop would
  remove objects outside the schema (N15, `06=e7f3822281`). A test that
  makes another schema depend on its schema must drop those objects
  itself. The check found no such leak in the harness, driver, database,
  agentbootstrap, schema, lease, changestream, controllernode,
  eventsource and model state packages at 06, or in the harness, driver,
  database, agentbootstrap, dbaccessor and upgradedatabase packages at
  the restacked top `08=05c37d6e9f` (PostgreSQL, `-race`).
- Failure injection must still test the intended operation and rollback.
  A different SQL error during setup is not equivalent coverage.

## Next tasks and completion

1. The six repair batches and subsequent catalog/nullable-BOOLEAN fixes
   are integrated, and the latest sweeps passed. Retain the
   [parallel repairs handoff](../handoffs/2026-11-10-stage-6-parallel-repairs.md)
   (N18) and exact-revision results. Remaining coverage work includes
   06-schema-skip and explicit backend identity; 06-schema-skip is a
   separately scoped task, not a reason to repeat completed repair batches.
   Route new production-query defects to Stage 5 rather than combining
   them silently with fixture changes.
2. Use D42's runtime-entry package list in the detail file. Keep deferred
   PostgreSQL import/export (D40) distinct from blockers. Later gates use
   N16's tiers and levels (smoke before commits, full post-commit,
   readiness before a bootstrap run or milestone). 06-test-speed (N17)
   and 06-leaseexpiry-db are done; 06-test-levels remains.
3. Run gates as one `go test` invocation with `-p` up to 8 on the
   unit-test server profile (N17, OPERATIONS.md), on both backends, at
   the restacked revision; record selectors, tags, skips and evidence. A
   fixed issue remains "awaiting rerun" until tested. The failed sweep at
   `09=a3818782aa` is historical; the passing selections at
   `09=f3340c6710` supersede it without closing their recorded skip and
   deferred-coverage gaps. Revalidate changed layers after redistribution;
   do not describe the selected sweep as every Juju package passing.
4. The functional milestone also requires Stage 9 runtime evidence;
   passing a Stage 6 gate alone is not "functional".

## Stop conditions

Ask before changing production semantics, adding skips, weakening an
assertion, broadening the gate, or choosing a new driver/schema workaround.
For transaction changes, test retry behaviour and inspect the affected
callback for non-repeatable side effects; do not claim a repository-wide
retry audit from a small passing suite. Concurrency tests must obey the
repository's race, stress and watcher-readiness rules.
