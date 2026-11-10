# PostgreSQL PoC: Stage 6 worker W6, errors, triggers and order (2026-11-10)

## Objective

Repair the 16 failed leaf tests in seven packages recorded by the
2026-11-10 re-sweep at `09=a3818782aa`. All are test-side: SQLite error
text and catalog queries, SQLite trigger DDL, a predicate-free join, one
bind, one timestamp and one order assertion.

Follow the
[Worker protocol](2026-11-10-stage-6-parallel-repairs.md#worker-protocol)
for workspace, scope, test commands, commits and the report.

## Starting state

- The integrator gives you the worktree (`../juju-4-pg-w6`), branch
  (`pg-work/w6-errors-triggers`), base commit and `-p` budget. The base
  includes Step 0's `DropFailingTriggerSQL` in
  `domain/schema/testing/trigger.go`.
- Evidence: [06-compatibility.md](../stages/06-compatibility.md)
  (rows 06-errors, 06-X29, 06-fk-check, 06-order),
  [DIALECT.md](../DIALECT.md) and the sweep JSONL named in the
  [parallel-repairs handoff](2026-11-10-stage-6-parallel-repairs.md#starting-state).

## Work and boundaries

Upstream homes are `new` unless stated.

1. **`domain/network/state`, 5 tests.**
   - X20, 4 tests. Apply N20 with the `internal/database` helpers, and
     keep the constraint name or the wrapping context in the assertion
     where both backends report it:
     - `TestCannotAddEmptyProviderSubnet` (`subnet_test.go:697`, assert
       at :716), `TestCannotAddEmptyProviderLinkLayerDevice`
       (`linklayer_merge_test.go:318`) and
       `TestCannotAddEmptyProviderIPAddress` (:341, assert at :360)
       match `CHECK constraint failed: chk_provider_id_empty`. Use
       `IsErrConstraintCheck`; both backends name
       `chk_provider_id_empty`.
     - `TestCreateK8sServicesDuplicateApplication`
       (`cloudservice_import_test.go:99`, assert at :119) matches
       `UNIQUE constraint failed: k8s_service.application_uuid`; PG
       reports index `idx_k8s_service_application`. Use
       `IsErrConstraintUnique` and keep the `inserting services:` prefix.
   - Trigger, 1 test: `TestDeleteSpaceExposureFailureIdentifiesTable`
     (`space_remove_test.go:251`) creates a SQLite trigger in a loop over
     two tables and two operations (:269-298). Use
     `schematesting.FailingTriggerSQL` with `Event` set to the
     operation, and drop it with `DropFailingTriggerSQL` on each pass
     (the PostgreSQL function would otherwise already exist on the next
     pass, 42723). `s.query` already runs through `database/sql`. Keep
     every per-pass assertion.
2. **`domain/machine/state`, 3 tests, all in `reprovision_test.go`.**
   - 06-fk-check: `TestDetachLostMachineCloudInstanceAllowsRetiredUnitRemoval`
     (:319) runs `PRAGMA foreign_key_check` at :354. The PostgreSQL
     schema declares no `DEFERRABLE` foreign keys, so every statement
     is checked at once and the preceding calls would already have
     failed on a violation. Run the PRAGMA only when
     `schematesting.BackendForTest()` is `BackendDqlite`, with a comment
     giving that reason. Keep the rest of the test on both backends.
   - Side-effect trigger:
     `TestDetachLostMachineCloudInstanceErrorsWhenUnitDisappearsBeforeRename`
     (:386) creates `delete_reprovision_unit`, an `AFTER DELETE` trigger
     on `relation_unit` that deletes the unit (:392-398).
     `FailingTrigger` cannot express this. Add a test-local switch on
     `BackendForTest()` that creates the SQLite trigger or a PL/pgSQL
     function and trigger with the same `WHEN` condition, executed
     through `s.runQuery` (`database/sql`). Confirm the test still
     reaches `expected 1 row affected, got 0` (:402) for the same
     reason on both backends.
   - Abort trigger: `TestDetachLostMachineCloudInstanceRollsBack`
     (:1056) creates `fail_reprovision_detach`, `BEFORE UPDATE ON
     machine_cloud_instance WHEN NEW.instance_id IS NULL` (:1064). Use
     `FailingTriggerSQL` with `When`. Keep all rollback checks.
3. **`domain/access/state`, 2 tests.**
   - X29, `TestDeletePermissionsByGrantOnUUID` (:1433): the helper
     `ensurePermission` (:1561-1575) uses `LEFT JOIN
     permission_object_type AS pot` with no `ON` clause. The `WHERE`
     filters make it one access type times one object type. Express
     that explicitly (for example `CROSS JOIN`, or scalar subqueries
     for both ids), and keep one row per call. Note that the helper
     binds its `grantTo` parameter into `grant_on` and `grantOn` into
     `grant_to`; the caller at :1438 depends on that. Preserve the
     values written. Renaming the parameters is optional; mention it in
     the report.
   - X23, `TestReadUserAccessForTarget` (:315): the debug lookup at
     :337-341 compares the `TEXT` column with `grant_to = 123` (42883).
     Use the string `'123'` or a bound string.
4. **`domain/model/state/model`, 2 tests (X26).** `TestCreateModelAndUpdate`
   (:205, assert at :229) and `TestCreateModelAndDelete` (:232) match
   `model table is immutable, only insertions are allowed` as the whole
   message. PostgreSQL adds `ERROR:` and `(SQLSTATE P0001)`. Match the
   message itself unanchored (N20).
5. **`domain/relation/state`, 1 test (X27).**
   `TestGetGoalStateRelationDataForApplication` (:4142, fails at :4214)
   differs only in `Since`: the fixture time is
   `time.Now().UTC()` (:4160). Apply the protocol's timestamp rule.
   06-compatibility (06-order) earlier recorded inner-slice order
   differences here, which `SameContents` does not cover. After the
   fix, run the test with `-count=5` on PostgreSQL. If
   `EndpointIdentifiers` order varies, apply the order rule to the
   production query.
6. **`domain/unitstate/state`, 1 test (X17).**
   `TestGetEndpointsForUnpopulatedUnit` (`updateunitports_test.go:250`,
   assert at :263) compares the endpoint list in order. Production
   `getEndpoints` (`updateunitports.go:301-325`) has no `ORDER BY`,
   and its call sites (:156, :170, :187) use the result as a set. Use
   `tc.SameContents`. Apply the same change to
   `TestGetEndpointsForPopulatedUnit`, which 06-compatibility records
   as failing intermittently for the same reason.
7. **`domain` (root package), 2 tests (X32).** `TestStateBasePrepare`
   and `TestStateBasePrepareKeyClash` (`state_test.go:41-104`) query
   `sqlite_schema`. The two `TestType` declarations use columns `name`
   and `type`, and the first test expects `name` to be `"schema"`
   (:63). Replace the source with one that has both columns and a fixed
   first row on both backends, for example a derived table
   `(SELECT 'schema' AS name, 'table' AS type) AS t`, and keep the
   expected value. Keep the statement-cache assertions
   (`stmt1 == stmt2` at :68, and the key-clash error at :100).

## Tests

All packages finish in a few minutes. Run each package fully on both
backends with the protocol's commands. For item 5, add
`-run 'TestRelationSuite/TestGetGoalStateRelationDataForApplication'
-count=5` on PostgreSQL.

## Safety and stop conditions

- Do not change production code, schema or `domain/schema/testing`.
- Out of scope: 06-schema-skip (the blanket skip in `domain/schema`).
  The integrator dispatches it separately after this round.
- If a PL/pgSQL trigger body fails inside `StdTxn`, stop and report the
  exact error; the integrator then decides whether Step 0 needs a
  further helper.
