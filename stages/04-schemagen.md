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
- Status: implemented, in review. Subsequent fixes were incorporated
  during Stage 6; the original commit/object inventories and verification
  reports are [historical reference](../history/schema-generation.md).
- Current contracts:
  - The translator, trigger functions, ordering, checked-in diff overrides,
    freshness checks, embedding and backend selectors already exist.
    Work here should be a named correction, not a new implementation.
  - `ControllerDDL()` / `ModelDDL()` remain dqlite. Stage 8 must select
    backend-aware DDL for runtime callers. On the current top, D11's
    dialect interface lives in `core/database/dialect`; the historical
    `schema.PostgreSQL` enum was replaced by `dialect.PostgreSQL` in
    Stage 6. Check the branch API before moving a fix to an earlier stage.
  - D13 uses a plain BIGINT and owned sequence for change-log IDs. Allocate
    after the per-namespace lock; preserve explicit fixture IDs. Test
    blocked-writer visibility and rollback; sequence gaps are valid.
  - Shared lookup, binary-key and boolean corrections must carry matching
    export types and legacy import deltas (D27/D30). U12 changed both
    relation-status ID columns to integers; do not restore the old TEXT
    workaround from historical notes.
  - Current 4.1 beta schema edits require explicit recreation of disposable
    PoC databases. Historic 4.0 payload types remain supported upstream.
    The generator rejects `.PATCH.sql`; future shared upgrade-patch support
    needs explicit design rather than silently omitting patches.
  - PostgreSQL FKs remain non-deferrable. X21's model-import support is
    deferred under D40, not an implicit instruction to change all FKs.
- Verification for further changes: run affected generator/schema tests
  with actual PostgreSQL-path proof and required race/stress coverage;
  regenerate and check freshness. Include default-backend and legacy
  import/export tests when shared types change. Record revision-specific
  evidence using OPERATIONS.md; old package passes are not a current gate.
