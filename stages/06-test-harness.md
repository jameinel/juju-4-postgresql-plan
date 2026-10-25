# Stage 06: PG test harness for domain state tests

Branch `pg-poc/06-test-harness`, on `pg-poc/05-queries`.

- Goal: run selected domain state suites against PostgreSQL to flush out
  type-affinity and dialect bugs early.
- Steps: a PG-backed equivalent of `internal/database/testing` and
  `domain/schema/testing` that creates a throwaway schema per test, enabled
  by the test env var. Start with lease, changestream, controllernode,
  model, application and machine.
- Verification: the chosen suites pass on both backends; known failures are
  listed here. Verify a PostgreSQL run really reached PostgreSQL, for
  example by pointing `JUJU_POSTGRESQL_TEST_URL` at an unreachable host and
  checking that the suite fails to connect.
- Status: in progress. The harness works and the dialect refactor removed
  the `core/database/schema` import cycle. Several suites pass on both
  backends; the others have known PostgreSQL failures, listed in the
  [current handoff](../handoffs/2026-10-25-stage-6-remaining.md).
- Restart notes:
  - `JUJU_POSTGRESQL_TEST_URL` selects PostgreSQL; each namespace opened by
    the test suite gets a uniquely named throwaway schema which is dropped
    after the test (N9). Without the variable, tests continue to use dqlite.
  - Controller and model test suites select the matching generated DDL.
    The suite exposes `SchemaDialect()` (a `core/database/dialect`
    value, D11) for tests that build their own schemas or catalog queries.
  - PostgreSQL test connections set `TimeZone=UTC` with
    `postgresql.WithTimeZoneUTC`; Stage 8 must retain this for lease
    timestamp semantics.
  - Commits after the harness commit, all on this branch (oldest first):

    | Commit | Kind |
    |---|---|
    | `feat(testing): add PostgreSQL domain test harness` | harness |
    | `fix(database): bind controller node id as text` | latent (U3) |
    | `test(lease): seed migration test leases with bound timestamps` | latent test (U4) |
    | `test(schema): add backend-aware failing trigger helper` | portability |
    | `test(controllernode): make address projection tests backend neutral` | portability |
    | `test(changestream): insert controller node ids as text` | latent test (U5) |
    | `fix(schema): store random key bytes in blob columns` | latent (U8, X11) |
    | `fix(schema): prefer agent_version values in v_model_config` | bug (U7, X15) |
    | `fix(schema): declare the removal force flag as boolean` | latent (U9, X25) |
    | `test(database): skip txn query logging test on postgresql` | harness (D25) |
    | `test(schema): seed controller api port as text` | latent test (X23) |
    | `test(domain): quote the user table in test sql` | latent test (X8) |
    | `test(domain): use single-quoted string literals in test sql` | latent test (X6, U6) |
    | `test(domain): replace sqlite date functions in test sql` | latent test (X7) |
    | `fix(schema): declare relation status type ids as integers` | latent (U12, X23, D30) |
    | `test(crossmodelrelation): match secret consumer units without instr` | latent test (X30) |
    | `fix(database): move schema dialects to a leaf package` | harness (D11 amended) |
    | `test(domain): replace insert or ignore and replace in test sql` | latent test (X7) |

    The earlier env-isolation fix is folded into the harness commit, and
    the explicit `change_log` id fix (D13) into Stage 4's
    `feat(schema): generate and embed postgresql ddl`.

  - Passing on PostgreSQL in their latest runs: `domain/lease/state`,
    `domain/controllernode/state`, `domain/changestream/state`,
    `domain/objectstore/state`, `domain/provisioner`,
    `domain/modelmigration/state/controller`, `core/database/schema`,
    `core/database/dialect`, `internal/database`,
    `core/watcher/eventsource`, `domain/schema`.
  - `RAISE(ABORT, ...)` sites still to convert to `FailingTriggerSQL`:
    `domain/removal/state/model/relation_test.go`,
    `domain/removal/state/model/relationwithremoteofferer_test.go`,
    `domain/application/state/application_refresh_test.go` and
    `domain/machine/state/reprovision_test.go`. `internal/migration`
    `abort_test.go` creates an SQLite `BEGIN ... END` trigger too.
- Findings:
  - `internal/testhelpers` `OsEnvSuite` (part of `IsolationSuite`) clears
    environment variables that are not in its `testingVariables` list.
    `JUJU_POSTGRESQL_TEST_URL` was missing from it, so `DqliteSuite` and
    `BackendForTest` never saw it. Suites built on
    `coretesting.BaseSuite` kept the variable. Fixed in
    `fix(testing): keep postgresql test url through env isolation`.
  - Failure injection uses `schematesting.FailingTriggerSQL`, which emits
    SQLite or plpgsql trigger DDL. PostgreSQL does not allow subqueries in
    a trigger `WHEN`, so the condition is evaluated in the function body.
    The remaining direct `RAISE(ABORT, ...)` sites are listed under
    restart notes.
  - Domain errors are masked from driver errors, so tests cannot use
    `database.IsErrConstraint*` on errors returned by state methods; match
    message text that both backends share.
  - Portability (Stage 4 generator): the `change_log` serialising trigger
    replaced explicit ids, so the changestream prune tests, which insert
    ids from 1000, pruned the wrong rows. Fixed in pgschemagen (D13
    amendment) in `fix(schema): keep explicit change_log ids on
    postgresql`; the PostgreSQL schemas were regenerated.
  - Dqlite defects found here are in the
    [upstream fix register](../UPSTREAM-FIXES.md).
- Assumptions to verify:
  - Domain code tolerates SERIALIZABLE retries; no transaction body has
    non-idempotent side effects outside the DB.
  - SQLite type affinity hides a bounded number of type mismatches that
    this harness will surface.
