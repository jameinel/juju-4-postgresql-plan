# Stage 6: task and evidence detail

Read only the selected task, its X*/D* references and relevant evidence.
[Stage 6](06-test-harness.md) owns scope and gating; [OPERATIONS.md](../OPERATIONS.md)
owns commands and evidence requirements. File locations below are entry
points from the 2026-10-25 run; re-grep symbols rather than trusting lines.

## Acceptance gates (D42)

### Runtime-entry gate: before Stage 8 implementation

Complete X37/X38 with their focused regression evidence. Rerun and triage
unexplained status failures; resolve those that block bootstrap or runtime
work. A remaining unexplained failure is not automatically safe to defer.

Run both backend paths for the following areas at the restacked revision:

| Area | Package selection |
|---|---|
| Driver and errors | `internal/database/postgresql`, `internal/database/drivererrors` |
| Schema and harness | `core/database/dialect`, `core/database/schema`, `internal/database`, `internal/database/testing`, `domain/schema/...`, `generate/pgschemagen` |
| Bootstrap | `agent/agentbootstrap`, including `TestPostgresBootstrapSuite` on PG |
| Coordination | `domain/lease/state`, `domain/changestream/state`, `domain/controllernode/state`, `core/watcher/eventsource` |

Expand `domain/schema/...` into individual packages before running: do not
run PostgreSQL packages concurrently. Use backend-proof tests and inspect
skip counts; an empty selection or a silently skipped suite is not a pass.
Fixture packages without tests are exercised by their consumers, not
counted as standalone coverage. Record actual build tags to distinguish
SQLite from dqlite; Stage 9 must still validate the real dqlite agent.

### Functional gate: before Stage 9 acceptance/comparison

Also cover the chosen workload, application/machine/provisioner/status/
removal paths, access/key/secret paths, and the dedicated cross-model
regression. The outstanding repairs below remain required where they
exercise these paths; finishing the entry gate does not waive them.
Stage 9 supplies actual controller/lifecycle evidence in addition to unit
checks. Freeze its exact workload-specific package/selectors when the
workload is selected, rather than inventing a complete benchmark here.

### Evidence and scope

Retain known failures and not-rerun packages. PostgreSQL model import/export
and dqlite recovery are deferred, not reported as passing. Shared 4.1 fixes
still require upstream legacy import/export tests (D40).

Before each gate run, record exact selectors, per-package budgets and
approved skips. New blockers return to their owning task; do not expand
the run to every discovered package or silently weaken the gate.

## Bounded repair tasks

Owner is `pg-poc/06-test-harness` except production SQL (Stage 5) and shared
schema/generator fixes (route to the relevant earlier stage). Each dispatch
selects one issue and named files, not the entire table. Inspect fixtures
and consumers, preserve assertions, then test on both backends. Stop on
unexpected production semantics or unapproved skipping. Classify each
change individually; backend-specific trigger syntax is portability, not
a blanket latent test defect.

| ID | Scope / required result | Readiness |
|---|---|---|
| 06-X23 | `domain/removal/state/model/secret_test.go`: bind revision IDs as strings to `secret_value_ref` and `secret_deleted_value_ref` TEXT columns. | Defined substitution; verify affected tests. |
| 06-X25 | `domain/provisioner/state/model/state_test.go`: boolean fixtures for `allocate_public_ip` and `read_only`; removal-model `state_test.go`: boolean `force`. | Inspect column/driver representation on both backends, then preserve logical fixture values. |
| 06-X27 | Removal schedule tests and `domain/macaroon/state`: use microsecond-representable times for round-trip assertions. X28: replace the out-of-range max-int sentinel with a representable bound. | Preserve boundary/ordering coverage; do not merely relax comparisons. Ask if production precision semantics must change. |
| 06-X16 | `domain/application/state/application_test.go`, `TestEnsureApplicationOnMultipleMachines`: `COUNT(*), uuid` without grouping. | Clarify intended assertion before choosing grouping versus separate queries. |
| 06-X9 | Same file: `CONCAT('application_', ?)` lacks PG parameter type context. | Choose a portable typed expression or bind the full fixture name; preserve the value. |
| 06-DDL | Application `application_test.go` / `unit_test.go`: test `charm_external_reference.created_at DATETIME`. | Use supported test DDL (`TIMESTAMP` proposed); verify time decoding. |
| 06-trigger-drop | Application `model_test.go`, `TestIsControllerModelControllerModel`: PG requires `DROP TRIGGER ... ON model`. | Use the harness dialect when a shared form is unavailable; verify both paths. |
| 06-errors | Machine FK-error assertion; model/model immutable-row assertions. | Keep typed/domain errors where exposed, otherwise match the specific message in both formats (X20/X26). |
| 06-failure-injection | Removal `relation_test.go`, `relationwithremoteofferer_test.go`; machine `reprovision_test.go`; application `application_refresh_test.go`: use `schematesting.FailingTriggerSQL`. | Preserve failing operation, condition and rollback checks. Approve helper extension before implementation. |
| 06-X29 | Access `permission_test.go`, `ensurePermission`: predicate-free `LEFT JOIN`. | Determine intended fixture cardinality before selecting the replacement join. |
| 06-U5 | `domain/upgrade/state/state_test.go`: bind node IDs as text. | Defined substitution; preserve IDs and assertions. |
| 06-fk-check | Machine `reprovision_test.go`: SQLite `PRAGMA foreign_key_check`. | Limit only the backend-specific assertion; confirm retained PG coverage before a whole-test skip (N10). |
| 06-status-diagnosis | `domain/status/state/model`: 48 recorded failures, not yet diagnosed after latest fixes. | Rerun first; report root cause, affected contract and owning stage before proposing fixes. |

PostgreSQL import-specific work is deferred: X21 deferral, modelimport
`typeof` checks, and the SQLite trigger in `internal/migration/abort_test.go`.
Do not add skips to upstream tests or silently discard these findings.
If any proves to be on a required runtime path, reconsider its scope with
the user instead of relying on the label "import".

## Latest recorded evidence

Snapshot from the last implementation handoff, not tests run during plan
refinement. `06=eac393da16`, `05=d658b402a6`, `08=2e67edc999`.
The `restack/` run used that 06 tip; the status row is from an older run.
See [history](../history/compatibility.md#historical-runs-and-restacking)
for logs. A missing command/build tag must be recovered before using a
historical pass to accept a new gate.

| Package | PG failures | Disposition |
|---|---:|---|
| `domain/objectstore/state` | 0 | Recorded pass. |
| `domain/keymanager/state` | 0 | 2026-10-26 at `08=7dc5b5bbc9`, full package; was 6 (X37, fixed). |
| `domain/storageprovisioning/state` | 0 | 2026-10-26 at `08=7dc5b5bbc9`, full package (659 s). |
| `domain/crossmodelrelation/state/model` | 23 | 2026-10-26 at `08=7dc5b5bbc9`. X37 offer-scope tests pass. Remaining, all test-side or U11: 16 fixtures bind integer `0` to `relation_status.updated_at` (mainly `package_test.go` `addOfferConnection`); 4 X25 `auto_prune` integer fixtures; 2 `instr` (X30, U11 secrets queries); 1 test `ORDER BY rowid` (`offer_test.go`, X31). After U11 was cherry-picked, `TestModelSecretsSuite` passes 29/29 at `08=c5f185e6f6`, so the 2 U11 failures are fixed. |
| `domain/crossmodelrelation` (U11 watcher tests) | 0 | 2026-10-26 at `08=c5f185e6f6`: `TestWatcherSuite/TestWatchRemoteConsumedSecretsChanges*` (2 tests). |
| `domain/secret/state` (consumed-secret tests only) | 3 of 5 | 2026-10-26 at `08=c5f185e6f6`, `-run 'TestStateSuite/.*Consumed'`. The 3 failures are in fixture `UpdateSecret` (X16 `updateSecret`, 42803 on `rp.policy`), not the U11 queries. Full package not run. |
| `internal/migration` | 3 | X21 (2), test trigger (1); PostgreSQL import deferred. |
| `domain/provisioner/state/model` | 5 | X25 fixtures. |
| `domain/provisioner` | 0 | Recorded pass. |
| `domain/modelmigration/state/controller` | 0 | Recorded pass; not proof of full migration support. |
| `domain/removal/state/model` | 50 | Reported breakdown: 25 X27, 22 X23, 3 X38. Other fixture sites remain listed above; rerun to reconcile. X38 fixed: `TestStorageSuite/TestGetDetachInfo` (5 tests) passes at `08=7dc5b5bbc9`; full rerun is part of the D42 gate. |
| `domain/status/state/model` | 48 | Older `x23/` run; rerun and diagnose. |
| `core/database/schema`, `core/database/dialect`, `internal/database`, `core/watcher/eventsource`, `domain/schema` | 0 | Recorded passes. |
| `agent/agentbootstrap` PostgreSQL bootstrap suite | not recorded here | Latest handoff reports passing; recover selector/revision/log or rerun. Supersedes Stage 8's old operation-4 failure. |

Earlier passes were reported for `domain/lease/state`,
`domain/controllernode/state` and `domain/changestream/state`. They do not
replace reruns after later shared changes.

Not rerun after the latest X6/X7/X8 repairs:

- `agent/agentbootstrap`, `domain/access/state`, `domain/modeldefaults/state`
- `domain/modelmigration/state/model`, `domain/model/state/controller`,
  `domain/model/state/model`, `domain/upgrade/state`
- `domain/operation/state`, `domain/cloud/state`, `domain/sequence/state`,
  `domain/machine/state`, `domain/application/state`
- `domain/modelconfig/state`, `domain/controllernode/state`,
  `domain/changestream/state`, `domain/lease/state`, `domain/macaroon/state`
- `domain/modelimport` (PostgreSQL execution deferred)

A package's presence in this evidence inventory does not automatically
make it a prerequisite for every milestone. Apply the D42 entry list above;
record other packages' disposition in the functional gate when its exact
workload is selected.
