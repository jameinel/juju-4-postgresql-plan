# PostgreSQL PoC: Stage 6 worker W2, removal model state (2026-11-10)

## Objective

Repair the 47 failed leaf tests in `domain/removal/state/model` recorded
by the 2026-11-10 re-sweep at `09=a3818782aa`. All are test-side; the
earlier sweep at `08=c5f185e6f6` (1333 s, 326 pass) found the same set.

Follow the
[Worker protocol](2026-11-10-stage-6-parallel-repairs.md#worker-protocol)
for workspace, scope, test commands, commits and the report.

## Starting state

- The integrator gives you the worktree (`../juju-4-pg-w2`), branch
  (`pg-work/w2-removal`), base commit and `-p` budget. The base includes
  Step 0's `DropFailingTriggerSQL` and `DropTriggerSQL` in
  `domain/schema/testing/trigger.go`; confirm before items 7 and 8.
- Evidence: [06-compatibility.md](../stages/06-compatibility.md),
  "Latest recorded evidence" row for this package, and the sweep JSONL
  named in the
  [parallel-repairs handoff](2026-11-10-stage-6-parallel-repairs.md#starting-state).

## Work and boundaries

Paths are relative to `domain/removal/state/model/`. Upstream homes:
`new` for every item unless you find an existing home in
UPSTREAM-FIXES.md; the trigger helper calls are `none`.

1. **X27, 25 tests: schedule time round trips.** Each schedules a
   removal with a nanosecond `time.Time` and compares the stored time:
   - `Test<Entity>RemovalNormalSuccess` and `...NotExistsSuccess` for
     Application (`application_test.go:476,508`), Machine
     (`machine_test.go:1147,1181`), Model and ControllerModel
     (`model_test.go:499,536,574,611`), Relation (`relation_test.go:144,175`),
     RelationWithRemoteConsumer (`relationwithremoteconsumer_test.go:148,180`),
     RelationWithRemoteOfferer (`relationwithremoteofferer_test.go:166,198`),
     RemoteApplicationOfferer (`remoteapplicationofferer_test.go:196,229`)
     and Unit (`unit_test.go:382,505`);
   - `storage_test.go`: `TestScheduleStorageAttachmentRemovalSuccess`
     (:103), `TestStorageInstanceScheduleRemoval` (:444),
     `TestVolumeScheduleRemovalSuccess` (:564),
     `TestFilesystemScheduleRemovalSuccess` (:747),
     `TestFilesystemAttachmentScheduleRemoval` (:1084),
     `TestVolumeAttachmentScheduleRemoval` (:1246),
     `TestVolumeAttachmentPlanScheduleRemoval` (:1825).

   Create the schedule time microsecond-representable where the test
   makes it. One commit for the package is fine if the pattern is the
   same.
2. **X23, 11 tests: secret value refs.** The fixture helper at
   `secret_test.go:290-305` binds the integer loop counter `i` to
   `secret_value_ref.revision_id` and
   `secret_deleted_value_ref.revision_id`, which are `TEXT NOT NULL`
   (`0012-secret.sql:90,101`). pgx: `unable to encode 0 into text
   format`. Apply D55 first: check what production writes to
   `revision_id`. Affected: `TestDeleteUnitOwnedSecrets`,
   `TestDeleteApplicationOwnedSecrets`,
   `TestGetApplicationOwnedSecretRevisionRefs`,
   `TestGetUnitOwnedSecretRevisionRefs`, `TestDeleteSomeRevisions`,
   `TestDeleteAllRevisionsFromNil`, `TestDeleteAllRevisions`,
   `TestGetUserSecretRevisionRefs`, `TestDeleteUserSecretRevisionRef`,
   `TestDeleteApplicationOwnedSecretContentIsolation`,
   `TestDeleteUnitOwnedSecretContentIsolation`.
3. **X25, 2 tests: `removal.force`.** `state_test.go:88,129`
   (`TestGetAllJobsWithData`, `TestDeleteJob`) bind `0` to the boolean
   `force`. Bind `false`.
4. **`DATETIME` DDL, 2 tests.** `TestDeleteCharmReturnConstraintError`
   (`application_test.go:1232`, DDL at :1244) and
   `TestDeleteUnitWithDanglingCharmReference` (`unit_test.go:1069`, DDL
   at :1088) create ad hoc tables with `DATETIME` (42704). Use a type
   valid on both backends and keep what the test checks.
5. **X9, 2 tests: untyped `CONCAT` parameter.**
   `TestDeleteIAASApplication` (:612) and
   `TestDeleteIAASApplicationWithUnits` (:657) fail with 42P18 in
   fixture SQL `CONCAT('application_', ?)` at `application_test.go:1428,1440`.
   Build the value in Go, or cast the parameter.
6. **X16, 1 test.** `TestEnsureApplicationOnMultipleMachines`
   (`application_test.go:278,284`) runs
   `SELECT COUNT(*), uuid FROM unit WHERE life_id = ...` (42803). Query
   the UUIDs and check the length and value, so that the test still
   proves exactly one row.
7. **Schema trigger drop plus X25, 1 test.**
   `TestIsControllerModelControllerModel` (`model_test.go:778-781`)
   runs `DROP TRIGGER trg_model_immutable_update` (42601: PostgreSQL
   needs `ON model`), then `UPDATE model SET is_controller_model = 1`.
   Use `schematesting.DropTriggerSQL("trg_model_immutable_update",
   "model")` through `database/sql`, and bind or write `TRUE`.
8. **Failure injection, 2 tests.** Both create SQLite `BEGIN ... END`
   triggers:
   - `TestDeleteRelationRollsBackUnitState` (`relation_test.go:377-387`);
   - `TestEnsureRelationWithRemoteOffererNotAliveCascadeRollback`
     (`relationwithremoteofferer_test.go:118-124`, `WHEN` with a
     subquery, `DROP TRIGGER` at :155).

   Use `schematesting.FailingTriggerSQL` and `DropFailingTriggerSQL`.
   A `WHEN` subquery is valid in the PostgreSQL function's `IF`. Confirm
   that the rollback assertions still run.
9. **X20, 1 test.** `TestDeleteMachineWithForceDoesNotCheckChildMachines`
   (`machine_test.go:2423`) matches `FOREIGN KEY constraint failed`;
   PostgreSQL returns 23503 on `fk_machine_parent_parent`. Assert the
   class with `internal/database.IsErrConstraintForeignKey` on the
   unwrapped error, keeping the check that delete failed for this reason.

## Tests

This package is slow on PostgreSQL: 300–350 s with `-p 8` at
`08=eac5b916c7`, and up to 16 minutes earlier. Iterate with suite and
test selectors, for example
`-run '^TestApplicationSuite$/^TestDeleteIAASApplication'`.

For the final full-package runs, report in at least every two minutes:

```bash
timeout 26m go test -count=1 -race -p 1 -timeout 25m -v \
  ./domain/removal/state/model/ 2>&1 | tee ~/pg-poc-work/w2/removal-pg.log
timeout 16m make run-go-tests TEST_PACKAGES=./domain/removal/state/model/ \
  TEST_ARGS='-v -race -p 1 -count=1 -timeout 15m' \
  2>&1 | tee ~/pg-poc-work/w2/removal-dqlite.log
```

with the protocol's environment setup. A timeout is not a test failure;
report it and the last test reached.

## Safety and stop conditions

- Stop an item and report it if the stored timestamp is produced inside
  production code, if `revision_id` writers disagree with its `TEXT`
  type, or if dropping the immutable-model trigger leaves state that
  later tests see.
- Do not change removal production code, schema or `domain/schema/testing`.
