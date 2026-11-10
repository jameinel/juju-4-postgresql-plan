# PostgreSQL PoC: Stage 6 worker W3, application state (2026-11-10)

## Objective

Repair the 14 failed leaf tests in `domain/application/state` recorded
by the 2026-11-10 re-sweep at `09=a3818782aa`. All appear test-side.
Earlier plan rows placed a `DATETIME` and a trigger-drop failure here;
those are in `domain/removal/state/model` (W2) and do not fail here.

Follow the
[Worker protocol](2026-11-10-stage-6-parallel-repairs.md#worker-protocol)
for workspace, scope, test commands, commits and the report.

## Starting state

- The integrator gives you the worktree (`../juju-4-pg-w3`), branch
  (`pg-work/w3-application`), base commit and `-p` budget. The base
  includes Step 0's `DropFailingTriggerSQL` in
  `domain/schema/testing/trigger.go`; confirm before item 3.
- Evidence: [06-compatibility.md](../stages/06-compatibility.md) and the
  sweep JSONL named in the
  [parallel-repairs handoff](2026-11-10-stage-6-parallel-repairs.md#starting-state).

## Work and boundaries

Paths are relative to `domain/application/state/`. Upstream homes are
`new` unless you find an existing one; trigger helper calls are `none`.

1. **X27, 5 tests: status `since`.** Tests build a nanosecond `since`
   and compare the stored `updated_at`:
   - `TestCreateApplicationWithStatus` (`application_test.go:293`,
     fails at :339);
   - `TestUpdateCAASUnitStatuses` (`unit_test.go:207`),
     `TestAddIAASUnits` (:753), `TestAddIAASUnitWithSpaceConstraint`
     (:808) and `TestAddCAASUnits` (:869), all checked by
     `assertUnitStatus` (`unit_test.go:718-737`).

   Make each test's `since` microsecond-representable where the test
   creates it. Leave `assertUnitStatus` comparing exactly.
2. **Order, 3 tests.**
   - `TestInsertApplicationEndpointBindOneToBeta` (:257, fails at :285)
     and `TestInsertApplicationEndpointBindOneToBetaDefaultedGamma`
     (:307, fails at :339): the helper `fetchApplicationEndpoints`
     (`application_endpoint_test.go:1218-1228`) has `ORDER BY s.name`
     over a `LEFT JOIN`. SQLite sorts `NULL` first in ascending order;
     PostgreSQL sorts it last. This is not X17: there is an `ORDER BY`.
     Make the `NULL` position explicit in the helper (`NULLS FIRST`
     works on both; SQLite since 3.30), and check the sibling queries at
     :1266, :1299 and :1331. Record it as `Dialect: new` (NULL sort
     order) in the report.
   - `TestGetApplicationStorageDirectives` (`applicationstorage_test.go:123`,
     fails at :188): production `GetApplicationStorageDirectives`
     (`storage.go:109`) has no `ORDER BY` (X17). Read its callers before
     choosing `tc.SameContents`; report it if any caller needs an order.
3. **Failure injection, 2 tests.** SQLite `BEGIN ... END` triggers
   (42601):
   - `TestSetApplicationCharmResourceReconciliationRollsBack`
     (`application_refresh_test.go:557`, trigger at :590,
     `BEFORE UPDATE OF charm_uuid ON application`);
   - `TestUnsetExposeSettingsFailureIdentifiesQuery`
     (`exposed_test.go:694`): a loop over four exposure tables creates
     `fail_exposure_delete` at :717 and drops it at :731.

   Use `schematesting.FailingTriggerSQL` and `DropFailingTriggerSQL`
   through `database/sql`. `FailingTrigger.Event` accepts
   `BEFORE UPDATE OF charm_uuid`. In the loop, the PostgreSQL function
   must be dropped each time, or the next `CREATE FUNCTION` fails with
   42723. Keep the per-table error assertion and the unchanged-state
   check.
4. **X9, 1 test.** `TestCreateCAASApplicationResetsExistingSequence`
   (`application_test.go:4647`) fails with 42P18 in the helper SQL
   `CONCAT('application_', ?)` at :4688 and :4701. Build the namespace in
   Go, or cast the parameter.
5. **X20, 2 tests: unique constraint text.**
   `TestUpsertK8sServiceProviderIDCollision` (:1306) and
   `TestK8sServiceUniqueApplication` (:1323) match
   `UNIQUE constraint failed: k8s_service.<column>`. PostgreSQL returns
   23505 on `idx_k8s_service_provider` and `idx_k8s_service_application`.
   Assert with `IsErrConstraintUnique`, keeping the
   `updating cloud service provider ID:` context check in the first, and
   keep a check that identifies which constraint failed if one can be
   written for both backends.
6. **X16, 1 test.** `TestUpdateCAASUnitSetsFQDN` (`unit_test.go:379`,
   fails at :412): the test's `assertFQDN` query (:406-410) selects
   `fa.address, fa.scope_id, count(*)` without `GROUP BY` (42803). Query
   the rows and check there is exactly one, so that the repeated-update
   check at :418-421 still runs and still proves no duplicate.

## Tests

This package is the slowest on PostgreSQL (500–560 s with `-p 8`, and
16–17 minutes when packages ran one at a time). Iterate with selectors,
for example `-run '^TestApplicationStateSuite$/^TestK8sService'`.

For the final full-package runs, report in at least every two minutes:

```bash
timeout 26m go test -count=1 -race -p 1 -timeout 25m -v \
  ./domain/application/state/ 2>&1 | tee ~/pg-poc-work/w3/application-pg.log
timeout 16m make run-go-tests TEST_PACKAGES=./domain/application/state/ \
  TEST_ARGS='-v -race -p 1 -count=1 -timeout 15m' \
  2>&1 | tee ~/pg-poc-work/w3/application-dqlite.log
```

with the protocol's environment setup. A timeout is not a test failure;
report it and the last test reached.

## Safety and stop conditions

- Stop an item and report it if a status time is set inside production
  code, if a caller of `GetApplicationStorageDirectives` depends on its
  order, or if a trigger needs a feature `FailingTrigger` lacks.
- Do not change application production code, schema or
  `domain/schema/testing`.
