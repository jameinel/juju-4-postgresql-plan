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
- Status: reopened, in progress. The first pass claimed the suites passed
  on PostgreSQL, but they had silently run on SQLite (see findings).
- Restart notes:
  - `JUJU_POSTGRESQL_TEST_URL` selects PostgreSQL; each namespace opened by
    the test suite gets a uniquely named throwaway schema which is dropped
    after the test. Without the variable, tests continue to use dqlite.
  - Controller and model test suites select the matching generated DDL.
    The controller-node state suite uses the same backend selector.
  - PostgreSQL test connections set `TimeZone=UTC` with
    `postgresql.WithTimeZoneUTC`; Stage 8 must retain this for lease
    timestamp semantics.
  - Commits after the harness commit, all on this branch:

    | Commit | Kind |
    |---|---|
    | `fix(testing): keep postgresql test url through env isolation` | harness |
    | `fix(database): bind controller node id as text` | latent (U3) |
    | `fix(lease): make lease pinning and tests valid on postgresql` | portability, plus latent test (U4) |
    | `test(schema): add backend-aware failing trigger helper` | portability |
    | `test(controllernode): make address projection tests backend neutral` | portability |
    | `test(changestream): insert controller node ids as text` | latent test (U5) |
    | `fix(schema): keep explicit change_log ids on postgresql` | portability (pgschemagen, D13 amended) |

  - Passing on PostgreSQL and SQLite with `-race` (verified):
    `domain/lease/state`, `domain/controllernode/state`,
    `domain/changestream/state`.
  - Known failure: `internal/database` `TestMigrationSuite` builds its
    patches with `schema.New` (SQLite bookkeeping DDL, `AUTOINCREMENT`).
    The harness needs a dialect accessor so such tests can use
    `schema.NewForDialect`.
  - Next: `domain/model/state` (controller and model),
    `domain/application/state`, `domain/machine/state` and
    `domain/sequence/state`. Stage 8's seed query fix also touches
    `domain/sequence/state`.
  - `RAISE(ABORT, ...)` sites still to convert to `FailingTriggerSQL`:
    `domain/removal/state/model/relation_test.go`,
    `domain/removal/state/model/relationwithremoteofferer_test.go`,
    `domain/application/state/application_refresh_test.go` and
    `domain/machine/state/reprovision_test.go`.
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
