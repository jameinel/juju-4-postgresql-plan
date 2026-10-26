# Stage 06: compatibility harness and acceptance

Branch `pg-poc/06-test-harness`, on `pg-poc/05-queries`.

## Status and purpose

The harness is implemented; the leaf-dialect refactor resolved its schema
import cycle. Remaining work is test repair, diagnosis and verification,
not building another harness. PostgreSQL bootstrap is reported passing;
this does not establish that all suites or the runtime worker work.

Stage 6 discovers compatibility defects; fixes can belong on Stages 3–5.
It supports bounded milestone gates rather than becoming a requirement to
make every Juju package pass before any runtime work resumes.

## Required context

Read the selected task and relevant rows in
[06-compatibility.md](06-compatibility.md), its X*/D* references, and
[OPERATIONS.md](../OPERATIONS.md) for running tests. Read N6, N9, N10 and
D11 if changing the harness. Commit inventories and past failures are in
[history](../history/compatibility.md), not mandatory setup reading.

## Existing contracts

- `internal/database/testing` and `domain/schema/testing` select PG via
  `JUJU_POSTGRESQL_TEST_URL`; absent the variable, existing SQLite/dqlite
  behaviour remains. Prove selection after environment/harness changes.
- Each test namespace has a unique throwaway schema (N9), with the
  matching controller/model DDL, deterministic closure and cleanup.
- `SchemaDialect()` returns the leaf `core/database/dialect` interface.
  Do not import the schema engine back into the harness and recreate the
  cycle. Use dialect-aware helpers for unavoidable backend-specific DDL.
- PG connections use UTC, SERIALIZABLE and bounded pools (cap 3 per
  namespace, X22). Stage 8 must preserve these runtime requirements.
- Failure injection must still test the intended operation and rollback.
  A different SQL error during setup is not equivalent coverage.

## Next tasks and completion

1. Repair one named issue from the task table or diagnose the status
   failures. Route production-query work to Stage 5; do not combine it
   silently with fixture changes.
2. Use D42's runtime-entry package list in the detail file. Confirm test
   selectors/budgets and classify status failures before dispatching runs.
   Keep deferred PostgreSQL import/export (D40) distinct from blockers.
3. Run the approved gate sequentially on both backends at the restacked
   revision; record selectors, tags, skips and evidence. A fixed issue
   remains "awaiting rerun" until tested.
4. Stage 8 can proceed after that entry gate is satisfied (met on
   2026-10-26; see the detail file). The functional milestone also
   requires Stage 9 runtime evidence; passing this gate alone is not
   "functional".

## Stop conditions

Ask before changing production semantics, adding skips, weakening an
assertion, broadening the gate, or choosing a new driver/schema workaround.
For transaction changes, test retry behaviour and inspect the affected
callback for non-repeatable side effects; do not claim a repository-wide
retry audit from a small passing suite. Concurrency tests must obey the
repository's race, stress and watcher-readiness rules.
