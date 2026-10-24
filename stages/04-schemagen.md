# Stage 04: PG schema generation

Branch `pg-poc/04-schemagen`, on `pg-poc/03-driver`.

- Goal: generate `domain/schema/postgresql/` from the dqlite patches with a
  reusable tool.
- Steps:
  1. `generate/pgschemagen`: translate patch by patch (types, identity
     columns, time defaults, `IIF`, generated columns, quoting `"user"`).
     Per-patch override files hold hand-written translations.
  2. Ordering pass in `pgschemagen` (D10), run over the whole ordered
     statement list of each namespace after translation and overrides.
     PostgreSQL resolves `REFERENCES` targets and view sources at `CREATE`
     time; SQLite does not (ANALYSIS.md section 4).
     - Strip each `FOREIGN KEY` whose target table, or the unique index it
       relies on, does not exist yet, and emit it as `ALTER TABLE ... ADD
       CONSTRAINT ... FOREIGN KEY`.
     - Move each view whose sources do not exist yet.
     - Append relocated statements to the end of the patch that creates
       their last dependency, after that patch's seed rows, so the patch
       count matches dqlite.
     - Leave tables, indexes, triggers and seed rows in place. No trigger
       is defined on a view, and PL/pgSQL bodies are not resolved at
       `CREATE` time.
     - Fail generation on dangling references and view cycles.
     This is safe because `Schema.Ensure` applies all patches in one
     transaction and PostgreSQL DDL is transactional.
  3. Trigger translation in `pgschemagen` (D12): each `CREATE TRIGGER`
     becomes a PL/pgSQL function and trigger, with `RAISE EXCEPTION` and
     the `WHEN` condition in the body. A `BEFORE INSERT` trigger on
     `change_log` takes the per-namespace `pg_advisory_xact_lock` (D13).
  4. The Go-built triggers (`domain/schema/triggers.go`,
     `modeltriggers.go`, `triggergen` output) are translated by step 3
     from the dqlite patch list; `triggergen` is unchanged.
  5. Make the bookkeeping in `core/database/schema` dialect-aware
     (`query.go`: identity column, timestamps, placeholders) (D11).
  6. Add a backend-aware schema selector (D11). Stage 8 uses it in
     bootstrap, the tracker and the upgrade worker.
- Verification:
  - Unit tests for the ordering pass: a forward FK, a forward view, an FK
    whose target unique index comes later, a two-table FK cycle (both
    become `ALTER TABLE`), and failures on a dangling reference and a view
    cycle.
  - Apply controller DDL and model DDL to empty PG schemas.
  - Table, view, index and trigger counts match dqlite (allowing documented
    differences).
  - Seed rows match row for row.
- Status: done, in review. Code on `pg-poc/04-schemagen`, four commits:
  `feat(database): support postgresql schema bookkeeping`,
  `feat(schema): generate and embed postgresql ddl`,
  `fix(schema): correct lookup types and revision unit counts` (U1, U2)
  and `refactor(schema): remove redundant postgresql overrides`.
- Restart notes:
  - Done: generic translator, trigger functions, forward-FK/view ordering,
    checked-in unified-diff overrides, freshness checks, embedded backend
    selectors, dialect-aware bookkeeping and documentation. There are 61
    controller and 156 model generated patches (zero-based file names;
    schema bookkeeping versions remain one-based).
  - `ControllerDDL()` / `ModelDDL()` still select dqlite. Stage 8 must use
    `ControllerDDLForBackend` / `ModelDDLForBackend` at runtime; no callers
    were changed here. `schema.NewForDialect(schema.PostgreSQL, ...)` is
    used by the PostgreSQL selectors.
  - Verified against PostgreSQL 16.15 on `lxd36-pgtest:pg-poc`: fresh
    controller/model application, idempotent reapplication, hash mismatch
    detection, rollback, foreign-key column counts, seed rows, every view's
    queryability, trigger guards, change events and blocked-writer commit
    ordering. Seed timestamps are checked against the application interval
    rather than compared literally across separately applied databases.
  - `go test -race ./core/database/schema ./domain/schema/...
    ./generate/pgschemagen ./domain/modelimport/... ./domain/export/...`
    passes with `JUJU_POSTGRESQL_TEST_URL` set.
    Bounded stress with two processes completed six runs without failures;
    the two schemas left by termination of the final active runs were
    explicitly removed. Tests otherwise drop their throwaway schemas.
  - `go generate ./domain/schema/...` and
    `go generate ./generate/export ./generate/modelimport` leave no diff.
    Changed-package `golangci-lint` reports zero issues; `make pre-check`
    passes (its repository-wide static analysis is opt-in and was not
    enabled).
  - Next: user review.
  - At the user's request, incorporated the shared dqlite cleanup from
    the other context: eleven lookup-column type corrections, regenerated
    export types, explicit legacy numeric-string import conversions and
    revision-updater grouping by application. Regression coverage checks
    integer persistence and import/export, nullable IDs, invalid input and
    zero/multiple-unit application counts.
  - Removed obsolete PostgreSQL override pairs and the redundant hunks
    in retained overrides. Regeneration leaves the final PostgreSQL SQL
    unchanged; remaining overrides still handle dialect-specific cases.
  - These corrections edit the current 4.1 beta base schema. The supported
    4.0-to-4.1 path bootstraps a new controller and migrates models through
    the versioned import transform; historic 4.0 payload types are
    unchanged. Recreate existing beta PoC databases because dqlite patch
    hashes change. The generator intentionally fails on `.PATCH.sql`
    files, so support for future shared upgrade patches needs explicit
    follow-up rather than silently omitting them.
- Findings (for later stages):
  - Object counts, including the bookkeeping table/index and excluding
    SQLite internal objects and PostgreSQL constraint indexes:

    | Namespace | Tables | Views | Explicit indexes | SQLite / PG triggers |
    |---|---:|---:|---:|---:|
    | controller | 95 | 20 | 90 | 89 / 90 |
    | model | 251 | 59 | 193 | 291 / 292 |

    The extra trigger in each PostgreSQL schema is the D13 advisory-lock
    trigger. IDs are assigned after acquiring the transaction lock; gaps
    from the earlier identity default and rolled-back transactions are
    expected. Do not assume contiguous IDs.
  - `ANALYSIS.md` section 4 describes an older source tree: all 380 triggers
    here are Go-built (not SQL-file declarations); there are no generated
    columns, REAL columns, GLOB or GROUP_CONCAT in the source DDL. Declared
    types include 258 INT, 8 INTEGER, 38 DATETIME, 24 TIMESTAMP, 42 BOOLEAN
    and 2 BLOB columns. Some trigger names exceed PostgreSQL's 63-byte
    limit; the generator adds a stable hash suffix.
  - Eleven TEXT-to-integer foreign-key mismatches in dqlite were masked by
    SQLite coercion. The shared schema now corrects
    `cloud_credential.auth_type_id`,
    `controller_node_password.password_hash_algorithm_id`,
    `autocert_cache.encoding`, `model_life.life_id`,
    `secret_permission.scope_type_id`, `charm_config.type_id`,
    `machine.password_hash_algorithm_id`, `machine_platform.os_id`,
    `application_platform.os_id`, `application_agent.password_hash_algorithm_id`
    and `unit.password_hash_algorithm_id`. PostgreSQL translates these
    directly to BIGINT; no type-correction overrides remain.
  - Other overrides make predicate-free singleton joins explicit, aggregate
    the latest-login timestamp, implement D17, compare the TEXT relation
    status to `'1'`, and disambiguate duplicate foreign-key constraint names
    on `application_remote_consumer`. `relation_status_type.id` and
    `relation_status.relation_status_type_id` both remain TEXT, matching
    dqlite; Stage 6 should check parameter binding at those callers.
  - The ordering pass preserves patch count, leaves tables/seeds/triggers in
    place and appends deferred FKs/views after the last dependency's seeds.
    Unit tests also cover FK cycles, unique-index targets, dangling
    references, view cycles, malformed/stale diffs and literal preservation.
  - Stage 5's `PRAGMA defer_foreign_keys` sites remain a Stage 6 integration
    concern. PostgreSQL foreign keys are not made deferrable by this stage.
