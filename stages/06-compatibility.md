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

Expand `domain/schema/...` into individual packages before running. Since
N17, PostgreSQL packages may run in parallel (`-p` up to 8) against a
unit-test profile server; before N17 they had to run one at a time.
Use backend-proof tests and inspect
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
approved skips. New blockers return to their owning task; do not silently
weaken the gate. Gates use the N16 tiers and levels: the readiness level
runs tiers A and B in full; do not add packages outside them.

## Parallel repair run (2026-11-10, in progress)

Step 0 is complete locally: `06=f4d3cbb673` adds
`DropFailingTriggerSQL` and `DropTriggerSQL`; the restacked tips are
`07=de24e7d3c2`, `08=09=0bb336f089`. The first helper drops the
PostgreSQL trigger function as well as the trigger, allowing recreation;
the second preserves schema-owned functions (X46). Trigger DDL runs
through `database/sql`, not sqlair.

At `09=0bb336f089`, `go build ./...` passed. Full package runs with
`-race -count=1 -p 2 -timeout 120s -v` passed on PostgreSQL (untagged)
and real dqlite (`make run-go-tests`, `dqlite,libsqlite3`):

| Package | PG pass / fail / skip | dqlite pass / fail / skip |
|---|---|---|
| `domain/schema/testing` | 3 / 0 / 0 | 3 / 0 / 0 |
| `domain/controllernode/state` | 37 / 0 / 0 | 37 / 0 / 0 |

The new tests cover unconditional and conditional failure injection,
rollback, repeated drops, recreation and preservation of the schema
trigger function. An invalid-URL probe of
`TestTriggerSuite/TestFailingTrigger` failed with connection refused,
proving PostgreSQL selection. An initial SQLite fallback run also passed
all three helper tests. `make pre-check` passed with static analysis
disabled by the default target; this is not a full lint result.
Logs: `~/pg-poc-work/integrator/step0-{build,pg,dqlite,backend-probe,sqlite,pre-check}.log`.

All six workers are dispatched on GPT-6.1 Terra, each from
`0bb336f0895c27f3eaf2f497e9678f382c030509` in its assigned
`../juju-4-pg-wN` worktree. W5–W6 started after W1 completed and W3
returned its edits for integrator validation. At most four implementation
workers run together; PostgreSQL test invocations remain within N19.

W1's six commits are integrated onto Stage 6 (`06=601211fa8e`), with
`07=cfa63fe52a`, `08=09=33bbe9d88e`. `go build ./...` passed at the new
tip. Both-backend package reruns completed with `-race -count=1 -p 2`
and the handoff's 120 s test timeout:

| W1 package | PG pass / fail / skip | dqlite pass / fail / skip |
|---|---|---|
| `internal/changestream/stream` | 36 / 0 / 0 | 36 / 0 / 0 |
| `domain/ssh/state/model` | 30 / 2 / 0 | 32 / 0 / 0 |
| `internal/worker/changestreampruner` | 3 / 0 / 0 | 3 / 0 / 0 |
| `internal/worker/dbrepl` | 1 / 0 / 19 | 20 / 0 / 0 |
| `internal/worker/dbreplaccessor` | 5 / 0 / 4 | 9 / 0 / 0 |

The PostgreSQL command exited 1 solely for the two reported SSH X18
failures; real dqlite exited 0. The earlier intermittent stream shutdown
failure did not recur, but is retained as a finding. Logs:
`~/pg-poc-work/integrator/w1-{pg,dqlite,build}.log`.
A bounded real-dqlite stress run of
`TestStreamSuite/TestConcurrentWritesAreStreamedAsLatestRows` completed
20 runs with zero failures before the 31 s outer timeout (exit 124);
this is bounded stress evidence, not an ordinary zero-exit test run.
Logs: `w1-stress-build.log`, `w1-dqlite-stress.log` in that directory.

Source commits:
`aeaf9f7019`, `fe0797778e`, `b9481524e4`, `678203c560`, `828c368e3f`,
`e6585b9fb4`. Only test files in the assigned packages changed.
The approved REPL suites are skipped only on PostgreSQL; their manifold
coverage remains enabled.
The clean W1 worktree and branch were removed after integration.

W1 exposed two SSH X18 production failures and one intermittent stream
shutdown failure; see
[the upstream register](../UPSTREAM-FIXES.md#production-findings-from-parallel-repairs-2026-11-10).
No production fix is approved or included.

W3's frozen six-file application working tree, based on `09=0bb336f089`,
passed full-package integrator validation on both backends: 632 suite
methods passed, none failed or skipped, with `-race -count=1 -p 1`.
PostgreSQL took 1133.253 s and real dqlite took 322.450 s. Logs:
`~/pg-poc-work/w3/parent-application-{pg,dqlite}.log`. Root-cause commits
and post-integration verification are pending.

W2's full real-dqlite run completed with two constraint-assertion
failures, in `TestDeleteCharmReturnConstraintError` and
`TestDeleteMachineWithForceDoesNotCheckChildMachines`; PostgreSQL
reproduced both and is still running the rest of the package.
The worker is investigating these assertion changes before commits.
Logs: `~/pg-poc-work/w2/parent-removal-{pg,dqlite}.log`.

W4's eight packages all passed on real dqlite at its frozen working
tree (HEAD `4f22469897` plus uncommitted edits):

| Package | dqlite pass / fail / skip |
|---|---|
| `domain/status/state/model` | 176 / 0 / 0 |
| `domain/secretbackend/state` | 53 / 0 / 0 |
| `domain/secret/state` | 142 / 0 / 0 |
| `domain/macaroon/state` | 15 / 0 / 0 |
| `domain/operation` | 11 / 0 / 0 |
| `domain/operation/state` | 159 / 0 / 0 |
| `domain/modelmigration/state/model` | 41 / 0 / 0 |
| `domain/secret` | 21 / 0 / 0 |

The first PostgreSQL run passed secretbackend, macaroon, operation and
modelmigration; status, secret state, operation state and secret exceeded
the 120 s package budget while progressing. These timeouts are
inconclusive, not regression failures. Exhaustive smaller PostgreSQL
groups, including two rounds for timestamp packages, are running.
Logs: `~/pg-poc-work/w4/parent-all-{pg,dqlite}.log` and
`parent-pg-groups/`.

W6's initial runs also need smaller groups: PostgreSQL network, machine,
relation and unitstate exceeded the short package budget; real dqlite
network and relation did likewise. Access, model state and the domain
root passed on both backends; unitstate passed on dqlite. Two machine
tests failed on dqlite (`CleansFilesystemAndPreservesIntent` and
`CleansVolumeAndPreservesIntent`), under investigation rather than
attributed to production yet. Logs:
`~/pg-poc-work/w6/parent-all-{pg,dqlite}.log`.

W5 edits are ready; the resource-list ordering assertion is being
restored pending review of its documented index-association contract.
No worker repair outside W1 is integrated yet.
No branches have been pushed; bootstrap and cleanup remain deferred
under N22.

## Bounded repair tasks

Owner is `pg-poc/06-test-harness` except production SQL (Stage 5) and shared
schema/generator fixes (route to the relevant earlier stage). Each dispatch
selects one issue and named files, not the entire table. Inspect fixtures
and consumers, preserve assertions, then test on both backends. Stop on
unexpected production semantics or unapproved skipping. Classify each
change individually; backend-specific trigger syntax is portability, not
a blanket latent test defect.

Before changing a fixture's binding type, confirm that the declared column
type correctly models the domain (D55). Matching an incorrect declaration
can hide the production defect rather than repair the fixture. Route shared
SQL/schema corrections to their owning stage; ask if this conflicts with a
task's test-only scope. Correct textual representations, including U3/U5's
unsigned node IDs, still require textual fixture binds.

**2026-11-10 assignment.** The re-sweep failures are split by package
into six worker handoffs under the
[parallel repairs handoff](../handoffs/2026-11-10-stage-6-parallel-repairs.md)
(N18). Workers own packages, not rows, so one row can span several
workers:

| Worker | Rows |
|---|---|
| W1 stream fixtures | 06-dialect-fixtures (stream, ssh model, pruner; dbrepl and dbreplaccessor skips, N21) |
| W2 removal model | 06-X23, 06-X25 (`force`), 06-X27 (schedules), 06-X16, 06-X9, 06-DDL, 06-trigger-drop, 06-failure-injection, 06-errors (FK) |
| W3 application | 06-X27 (`since`), 06-order (NULL sort, storage directives), 06-failure-injection, 06-X9, 06-errors (unique), 06-X16 |
| W4 timestamps and `%q` | 06-X27 (status, secretbackend, secret, macaroon, operation), X28, 06-unknown-column (operation, modelmigration), 06-order (secret watcher), 06-status-diagnosis |
| W5 bind types | 06-bind-types, 06-X25 (provisioner), 06-U5, 06-upgradesteps, 06-cmr-updated-at, 06-unknown-column (crossmodelrelation `rowid`), 06-errors (resource FK), 06-order (resource, cmr ingress) |
| W6 errors, triggers and order | 06-constraint-errors, 06-errors (model), 06-failure-injection and 06-fk-check (machine), 06-X29, 06-dialect-fixtures (`domain` root), 06-order (relation, unitstate) |

The integrator updates each row's revision and evidence when it
integrates a worker. 06-X17 is not in the re-sweep failures; 06-test-levels,
06-schema-skip and `domain/recovery/state` are outside this round.

| ID | Scope / required result | Readiness |
|---|---|---|
| 06-X23 | `domain/removal/state/model/secret_test.go`: bind revision IDs as strings to `secret_value_ref` and `secret_deleted_value_ref` TEXT columns if TEXT is the intended domain representation. | Confirm the representation against production writers/consumers first (D55), then verify affected tests. If the schema is incorrect, stop and scope its correction rather than masking it with a fixture conversion. |
| 06-X25 | `domain/provisioner/state/model/state_test.go`: boolean fixtures for `allocate_public_ip` and `read_only`; removal-model `state_test.go`: boolean `force`. | Inspect column/driver representation on both backends, then preserve logical fixture values. |
| 06-X27 | Removal schedule tests, `domain/status/state/model` (48 `Since` round trips) and `domain/macaroon/state`: use microsecond-representable times for round-trip assertions. X28: replace the out-of-range max-int sentinel with a representable bound. | Preserve boundary/ordering coverage; do not merely relax comparisons. Ask if production precision semantics must change. |
| 06-X16 | `domain/removal/state/model/application_test.go`, `TestEnsureApplicationOnMultipleMachines`: `COUNT(*), uuid` without grouping. `domain/application/state/unit_test.go`, `TestUpdateCAASUnitSetsFQDN`: `fa.address, fa.scope_id, COUNT(*)` without grouping. | Preserve the count and selected values; the FQDN test must also reach its repeated-update/idempotence check. Clarify the assertions before choosing grouping versus separate queries. |
| 06-X9 | `CONCAT('application_', ?)` lacks PG parameter type context in both `domain/application/state/application_test.go` and `domain/removal/state/model/application_test.go`. | Choose a portable typed expression or bind the full fixture name; preserve the value. |
| 06-DDL | Removal model `application_test.go` (`TestDeleteCharmReturnConstraintError`) and `unit_test.go` (`TestDeleteUnitWithDanglingCharmReference`): ad hoc tables with `DATETIME` (42704). Corrected 2026-11-10; earlier text placed these in application state. | Use supported test DDL (`TIMESTAMP` proposed); verify time decoding. W2. |
| 06-trigger-drop | Removal model `model_test.go`, `TestIsControllerModelControllerModel`: PG requires `DROP TRIGGER ... ON model`. Corrected 2026-11-10; earlier text placed this in application state. | Use `schematesting.DropTriggerSQL` (Step 0 of the parallel repairs handoff); verify both paths. W2. |
| 06-errors | Removal model FK assertion (`TestDeleteMachineWithForceDoesNotCheckChildMachines`), application unique-index assertions, resource FK assertion, model/model immutable-row assertions. | Test-only (N20): use the `internal/database` constraint helpers where the error is a constraint violation, otherwise match the specific message unanchored (X20/X26). W2, W3, W5, W6. |
| 06-X17 | `domain/model/state/controller/state_test.go`, `TestGetActivatedModelUUIDs`: assert the returned UUIDs with `tc.SameContents`, not by index. The query has no `ORDER BY`, and its only caller (`getWatchActivatedModelsMapper`) uses the result as a set. | Defined; test-only. Passed in the 2026-11-10 re-sweep; not assigned. |
| 06-failure-injection | Removal `relation_test.go`, `relationwithremoteofferer_test.go`; machine `reprovision_test.go`; application `application_refresh_test.go` and `exposed_test.go`; network `space_remove_test.go`: use `schematesting.FailingTriggerSQL`. | Preserve failing operation, condition and rollback checks. Loops drop the trigger with `DropFailingTriggerSQL` on each pass. The machine side-effect trigger gets a test-local backend switch (N18 handoff). W2, W3, W6. |
| 06-X29 | Access `permission_test.go`, `ensurePermission`: predicate-free `LEFT JOIN`. | Determine intended fixture cardinality before selecting the replacement join. |
| 06-U5 | `domain/upgrade/state/state_test.go`: bind node IDs as text. | Defined substitution; preserve IDs and assertions. |
| 06-fk-check | Machine `reprovision_test.go`: SQLite `PRAGMA foreign_key_check`. | Run the assertion on dqlite only: the PostgreSQL schema has no deferrable foreign keys, so a violation fails at the statement. Keep the rest of the test on both backends (N10). W6. |
| 06-status-diagnosis | `domain/status/state/model`: 48 failures, diagnosed 2026-10-26 at `08=c5f185e6f6`: all 48 are X27 round-trip precision assertions (`Since` nanoseconds versus PG microseconds; 27 via `assertStatusInfoEqual`, the rest `DeepEquals` on `Since`). No SQL errors; 125 pass. Production status code has no in-memory comparison of stored timestamps (status history filtering reads the log file). | Not a runtime blocker. Repair with 06-X27: microsecond-representable fixture times, preserving the assertions. |
| 06-schema-skip | `domain/schema` `schemaBaseSuite.SetUpTest`: one blanket `SkipOnPostgreSQL` (added in `09b78ef3a2`, never approved) skips 105 trigger and schema leaf tests on PG at `09=a3818782aa` (93 at the earlier gate). Make them backend-aware: select DDL with `ModelDDLForBackend`/`ControllerDDLForBackend`, make assertions dialect-neutral, and limit skips to genuinely SQLite-only checks with N10 reasons. | Probe at `08=c5f185e6f6` (temporary edit, reverted): trigger suites 51 pass, 2 upstream skips, 8 fail (7 SQLite `datetime()` fixtures, 1 SQLite error text); schema suites 9 pass, 2 upstream skips, 21 fail (`sqlite_master` introspection, SQLite error-text regexes where guard triggers raise the right message, `typeof()`, bool view column scanned as int, SQLite-only DDL in `storage_test.go`). No production defect found. Logs: session `files/d42/schemaprobe/`. Out of scope for the 2026-11-10 parallel round; a separate optional task. |
| 06-test-levels | Script the N16 levels: smoke (fixed core plus the domains touched by a diff), full and readiness (tiers A and B), each run on both backends with per-package logs, pass/fail/skip counts and a known-failure list for readiness. | Defined; selection from N16. Check the tier lists against `go list` when run, so new packages are not missed. |
| 06-test-speed | Cut the per-test PostgreSQL setup cost and allow parallel PostgreSQL packages without changing production `Ensure` (moved from Stage 11). See [06-test-speed measurements](#06-test-speed-measurements). | Parallel packages done (N17, `08=eac5b916c7`): unit-test profile plus a per-process D53 key; see [06-test-speed result](#06-test-speed-result). Per-test cost is unchanged (~0.5 s). Optional follow-ups: move schema drop off the test's critical path; split `domain/application/state` and `domain/removal/state/model`, which bound `-p 8` wall time. A database pool was measured and gives no gain ([database pool PoC](#database-pool-poc), N17). |
| 06-leaseexpiry-db | `internal/worker/leaseexpiry/store.go` runs its own SQL. Exercise expiry against a real controller schema on both backends. | Done in Stage 5 `76aac853dd`: `TestDatabaseStoreSuite/TestExpireLeases` covers expired, future, null-expiry and pinned rows. Full affected packages pass with `-race` on SQLite, real dqlite and PostgreSQL at `09=a3818782aa`; deterministic SQLite tests also cover skew and the equality boundary. |
| 06-dialect-fixtures | Test fixtures built for SQLite that fail on PG before reaching the code under test (sweep 2): `AUTOINCREMENT` in a schema DDL passed by `internal/changestream/stream`, `domain/ssh/state/model`, `internal/worker/dbrepl`, `dbreplaccessor`, `changestreampruner` and `core/database/schema` (`schema.go:42`; 97 failures); SQLite `CREATE TRIGGER` bodies using `OLD`/`NEW` (`domain/machine/state` 21, `domain/access/state`, `domain/recovery/state`, removal model); `sqlite_schema` in `domain`. | Find the shared fixture source first; most are one helper each. Use the harness dialect or `FailingTriggerSQL`; do not skip. The `domain/lease/state` `datetime()` fixtures were fixed in Stage 6 `ebba92514b`; the package passes on PostgreSQL at `09=a3818782aa`. |
| 06-bind-types | Test fixtures binding values the PG driver will not encode (sweep 2): integer to boolean (`domain/blockdevice/state` 6, `domain/storage/state` `persistent` in `base_test.go` `newPersistentModelVolume` and `in_use` in `import_test.go`, both fixtures, checked 2026-11-05); integer to text (`domain/controllerupgrader/state` 4, U5 `domain/upgrade/state`); empty string to bigint (`domain/resource/state` 3). Also X25 (provisioner, crossmodelrelation). | Check each against production first: blockdevice may be a production bind, not a fixture. |
| 06-constraint-errors | `domain/network/state`: PG 23514 (`chk_provider_id_empty`) and 23505 are not mapped to the domain errors the tests expect (8 failures in sweep 2; 4 X20 in the 2026-11-10 re-sweep, plus one trigger test). | Test-only (N20): the tests match SQLite message text, not domain errors. Use `IsErrConstraintCheck`/`IsErrConstraintUnique` and keep the constraint name or wrapping context where both backends report it. W6. |
| 06-unknown-column | Sweep 2 42703 failures: `domain/operation/state` (4), `domain/modelmigration/state/model` (1), `domain/crossmodelrelation` (1). | Triaged 2026-11-10, all test SQL: `%q` makes a string literal an identifier on PostgreSQL (X6, operation and modelmigration; W4), and `ORDER BY rowid` (X31, crossmodelrelation; W5, rewrite through the ordered getter). |
| 06-upgradesteps | `internal/worker/upgradedatabase/upgradesteps` (3 failures in the re-sweep and sweep 2). | Triaged 2026-11-10: fixtures bind `0` to boolean `cloud.skip_tls_verify` (X25); test-only. W5. |
| 06-cmr-updated-at | `domain/crossmodelrelation/state/model`: 16 fixtures bind integer `0` to `relation_status.updated_at` (mainly `package_test.go` `addOfferConnection`). | Bind a time value valid on both backends; confirm no production path writes an integer. |
| 06-order | Tests that depend on row order without `ORDER BY`: `domain/relation/state` goal state (`SameContents` used, but inner slices ordered), `domain/secret` `SecretTriggerSliceAssert`, and both `TestGetEndpointsForPopulatedUnit` / `TestGetEndpointsForUnpopulatedUnit` in `domain/unitstate/state/updateunitports_test.go`. X17 class. | The unitstate query has no ordering and its production caller consumes endpoints as map keys. Both assertions reproduced intermittently on PG; preserve membership and multiplicity with an order-independent check. 2026-11-10 triage: the relation goal-state failure is X27 (timestamp precision), not order; re-run it with `-count=5` after the fix to check inner-slice order (W6). Further cases: application endpoint helpers sort `NULL` differently under `ORDER BY` (`NULLS FIRST`, `Dialect: new`) and storage directives have no `ORDER BY` (W3); resource `ListResources` and cmr `GetRelationNetworkIngress` have no `ORDER BY` (W5; check callers, report any production need); secret revision-expiry changes (W4). |

PostgreSQL import-specific work is deferred: X21 deferral, modelimport
`typeof` checks, and the SQLite trigger in `internal/migration/abort_test.go`.
Do not add skips to upstream tests or silently discard these findings.
If any proves to be on a required runtime path, reconsider its scope with
the user instead of relying on the label "import".

## 06-test-speed measurements

2026-11-05, Stage 6 `7fe0fe0a2c`, PostgreSQL 16.15 in LXD on a 28-CPU
host, `fsync=off`, `full_page_writes=off`, `max_locks_per_transaction=64`
(shared lock table 8,768, X42). The test role has no `CREATEDB`, so
per-test databases or `CREATE DATABASE ... TEMPLATE` are unavailable.
`domain/unitstate/state`: 121 tests, 5 s on SQLite, 59 s on PostgreSQL
(median 0.47 s per test). One model-schema test, measured by phase with a
temporary test (deleted):

| Phase | Time |
|---|---:|
| Admin pool, `CREATE SCHEMA`, namespace pool | ~13 ms |
| `Ensure` of the model DDL (158 patches, ~2,440 statements) | 230–260 ms |
| Same DDL as one multi-statement `Exec` | 210–220 ms |
| N15 dependents check | 50–70 ms |
| `DROP SCHEMA ... CASCADE` | 115–130 ms |
| `TRUNCATE` of all 252 tables (52 hold seed rows) | ~100 ms |

Building schemas concurrently without the D53 lock: two in 244 ms, four
in 359 ms (against ~880 ms serially). Four concurrent drops failed with
53200 (out of shared memory).

Findings: DDL execution on the server, not round trips, dominates setup.
Truncate-and-reseed would save under half the cost and changes test
isolation. Teardown (check and drop, ~175 ms) can leave the test's
critical path. Building ahead or in parallel needs a lock table larger
than the default and a way past the per-database D53 lock for test
schemas. Earlier (2026-10-28): with the D53 lock, three packages in
parallel took 20% longer and small packages about twice as long; without
it, the same three packages peaked at about 80% of the lock table
([08-delete-locks result](08-dbaccessor.md#08-delete-locks-result)).

## 06-test-speed result

2026-11-05, `08=eac5b916c7`, server `lxd36-pgtest` with the unit-test
profile (N17: `max_locks_per_transaction=512`, about 70k lock slots).
16 tier A packages whose serial times sum to 197 s; one database, a
schema per test. "Per-process" is the N17 D53 key derived from the test
process PID; "shared" is the production `juju_ddl` key, which serialises
DDL across all test processes. Each run used one `go test` invocation.

| Run | Wall time |
|---|---:|
| Per-process key, `-p 1` | 208.4 s |
| Per-process key, `-p 4` | 121.1 s |
| Per-process key, `-p 8` | 112.5 s |
| Shared key, `-p 4` | 140.5 s |
| Shared key, `-p 8` | 137.1 s |

Per-package time, seconds:

| Package | `-p 1` | `-p 4` | `-p 8` | Shared `-p 4` | Shared `-p 8` |
|---|---:|---:|---:|---:|---:|
| `domain/network/state` | 84.9 | 92.9 | 98.8 | 96.8 | 116.2 |
| `domain/access/state` | 19.3 | 25.5 | 33.8 | 52.9 | 80.3 |
| `domain/agentpassword/state` | 18.2 | 23.6 | 31.4 | 30.0 | 60.5 |
| `domain/objectstore/state` | 12.1 | 14.9 | 19.8 | 27.9 | 44.3 |
| `domain/controllernode/state` | 7.3 | 9.8 | 14.2 | 17.9 | 45.5 |
| `domain/port/state` | 9.9 | 12.9 | 16.5 | 17.4 | 29.3 |

No run raised 53200. The same five failures (access and network state,
see the sweep 2 rows) occur in every run and predate this work.

Findings: wall time is bounded by the longest package (network, ~85 s
serial), so `-p 8` gains little over `-p 4` for this set; a full tier
A+B run should gain more. With the shared key, each package slows as
others' DDL queues behind the lock, which loses most of the parallel
gain. Per-test keys would add nothing: no tier A test calls
`Parallel()`, so one process builds one schema at a time.

### Database pool PoC

2026-11-05, experimental branch `pg-poc/experiment-db-pool`
(`2e04f6fdd4`, off `09=eac5b916c7`, not on the stack, published). The
branch holds the pool code and `scripts/postgresql-poc/db-pool-bench`
(scripts, package list, results). The pool gives
parallel packages separate catalogs. Each relation user may connect only
to its own database (`pg_hba`) and has no `CREATEDB`, so a pool of N
needs N data-integrators: `di01`..`di08`, co-located with `postgresql`
in `lxd36-pgtest:admin/pgtest`, databases `juju-01`..`juju-08`.
`JUJU_POSTGRESQL_TEST_URL_POOL` names a file of URLs, one per line; each
test process takes the lowest free slot with a non-blocking `flock` on
`<file>.slot-<n>.lock` and uses `urls[slot % N]` for its lifetime. The
pool takes precedence over `JUJU_POSTGRESQL_TEST_URL`.

Run: the 138 sweep 2 packages (117 `domain`, 18 `internal`, 2 `core`,
`apiserver/stateauthenticator`), untagged `go test -count=1 -p 8`,
warm build cache, one invocation per N. Sampler: connections per
database and host load every 5 s. The host has 28 CPUs.

| N | Wall time | Summed package time | `application/state` | Peak connections per database | Peak load |
|---:|---:|---:|---:|---:|---:|
| 1 | 591 s | 4309 s | 560 s | 22 | 16.3 |
| 4 | 566 s | 4036 s | 505 s | 10 | 13.6 |
| 8 | 612 s | 4078 s | 501 s | 5 | 17.5 |

Every run: 99 packages pass and 36 fail, the sweep 2 failures. Two
differ between runs. `domain/model/state/controller`
`TestGetActivatedModelUUIDs` failed at N=8 on result order (06-order).
`domain/status` failed at N=1 in schema teardown: the query listing
objects outside the test schema that depend on it raised `cache lookup
failed for attribute ... (XX000)` while other processes ran DDL in the
same database. That race needs a shared database; the pool avoids it.
If it recurs, retry or tolerate XX000 in that harness query.

Findings: the pool does not pay. Separate catalogs cut summed package
time by about 6% and wall time by 0–4%, within noise. At `-p 8` the
wall time is bound by `domain/application/state` (500–560 s, tests run
serially in one process); the next longest is
`domain/removal/state/model` (300–350 s). Against sweep 2 (about 58
minutes serial, one package at a time) `-p 8` on one database is about
6× faster. To go faster, cut per-test cost (schema setup and drop,
about 0.5 s per test) or split the longest packages; more databases
will not. The pool is not pursued (N17).

## PostgreSQL re-sweep

2026-11-10, 11:33:35–11:55:25 UTC, at `09=a3818782aa`, Go 1.27.3
linux/amd64, PostgreSQL 16.15 on the `unit-test` profile (`fsync=off`,
`full_page_writes=off`, `max_locks_per_transaction=512`,
`max_connections=100`). One invocation used
`-race -count=1 -p 8 -timeout 45m -v -json`, with no dqlite build tags
and a 50-minute outer timeout. It exited 1 after 21m50s, not due to a
timeout. No race detector reports appeared. This race-enabled run does
not establish whether an uninstrumented run meets the under-15-minute
target.

The selection is the 138-package pool-benchmark manifest at
`2e04f6fdd4:scripts/postgresql-poc/db-pool-bench/packages.txt`, plus the
eight packages below. It was checked against current `go list` results;
`domain/unitless/state` has tests but they use a nil state and execute no
SQL. The original manifest also includes test-only database users and
deferred import tests, so this is a re-sweep, not a readiness gate.

To reproduce the selection from the code repository, follow
[OPERATIONS.md](../OPERATIONS.md) for the profile, safety checks and
progress monitoring. A full sweep needs the shared server to itself
(N19). Set
`LOG_FILE` to a path outside both repositories:

```bash
: "${LOG_FILE:?Set LOG_FILE to a path outside both repositories}"
manifest=$(
  git --no-pager show \
    2e04f6fdd4d9c882aa6be46c764be1bf654c32d7:scripts/postgresql-poc/db-pool-bench/packages.txt
) || exit 1
mapfile -t packages < <(
  {
    printf '%s\n' "$manifest"
    printf '%s\n' \
      agent/agentbootstrap core/database/dialect generate/pgschemagen \
      internal/database/connsource internal/database/drivererrors \
      internal/database/postgresql internal/database/postgresql/testing \
      internal/worker/leaseexpiry
  } | sort -u
)
if [ "${#packages[@]}" -ne 146 ]; then
  printf 'Expected 146 packages, got %d\n' "${#packages[@]}" >&2
  exit 1
fi
unset JUJU_POSTGRESQL_TEST_URL_POOL
JUJU_POSTGRESQL_TEST_URL=$(cat "$HOME/.config/juju-pg-poc/db-url") || exit 1
if [ -z "$JUJU_POSTGRESQL_TEST_URL" ]; then
  printf 'PostgreSQL test URL is empty\n' >&2
  exit 1
fi
export JUJU_POSTGRESQL_TEST_URL
set -o pipefail
timeout 50m go test -json -v -race -count=1 -p 8 -timeout 45m \
  "${packages[@]/#/./}" 2>&1 | tee "$LOG_FILE"
```

Of 146 packages, 108 passed, 35 failed and three had no standalone
tests (`domain/schema/testing`, `internal/changestream/testing`,
`internal/database/testing`). Counting only leaf test outcomes, not
their parent suites, gives 5,139 passes, 333 failures and 170 skips.
The skips include 105 schema tests behind the blanket PG guard
(06-schema-skip), 17 dqlite-only `internal/database` tests, 32
dqlite-only dbaccessor tests and one requiring a dqlite server, 10
`internal/migration` tests and five others. They are coverage gaps, not
passes.
`domain/lease/state` and `internal/worker/leaseexpiry` passed; the
PostgreSQL bootstrap and driver packages passed. Before the run the
database had no other sessions or test/controller/model schemas; no such
schemas remained after the sweep or focused rerun.

| Failing package | Failing leaf tests |
|---|---:|
| `domain` | 2 |
| `domain/access/state` | 2 |
| `domain/application/state` | 14 |
| `domain/blockdevice/state` | 6 |
| `domain/controllerupgrader/state` | 4 |
| `domain/crossmodelrelation/state/model` | 21 |
| `domain/macaroon/state` | 6 |
| `domain/machine/state` | 3 |
| `domain/modelimport` | 9 |
| `domain/modelimport/state/model` | 3 |
| `domain/modelmigration/state/model` | 1 |
| `domain/model/state/model` | 2 |
| `domain/network/state` | 5 |
| `domain/operation` | 1 |
| `domain/operation/state` | 5 |
| `domain/provisioner/state/model` | 5 |
| `domain/recovery/state` | 11 |
| `domain/relation/state` | 1 |
| `domain/removal/state/model` | 47 |
| `domain/resource/state` | 5 |
| `domain/secret` | 1 |
| `domain/secretbackend/state` | 11 |
| `domain/secret/state` | 8 |
| `domain/ssh/state/controller` | 1 |
| `domain/ssh/state/model` | 32 |
| `domain/status/state/model` | 49 |
| `domain/storage/state` | 8 |
| `domain/unitstate/state` | 1 |
| `domain/upgrade/state` | 1 |
| `internal/changestream/stream` | 36 |
| `internal/migration` | 3 |
| `internal/worker/changestreampruner` | 3 |
| `internal/worker/dbrepl` | 19 |
| `internal/worker/dbreplaccessor` | 4 |
| `internal/worker/upgradedatabase/upgradesteps` | 3 |

Most failures fall under the existing fixture, bind, precision, ordering,
constraint and unknown-column tasks above. Import remains deferred.
The 2026-11-10 triage for the parallel repairs handoff classed the
unknown-column, constraint and upgrade-step failures as test-side (see
their rows); `domain/recovery/state` is not yet triaged. Two findings
need particular attention:

- `domain/unitstate/state` previously passed but
  `TestGetEndpointsForUnpopulatedUnit` failed on row order. Its query has
  no `ORDER BY`; the production caller uses the endpoints as map keys.
  A focused `-race -count=10` rerun of both populated and unpopulated
  tests had three failures in 20 PostgreSQL cases (one populated, two
  unpopulated); all 20 SQLite cases passed. Track both assertions under
  06-order rather than assuming insertion order.
- The observed SQLSTATE 42803 failures were in test assertions:
  `TestEnsureApplicationOnMultipleMachines` in removal model, and
  `TestUpdateCAASUnitSetsFQDN` in application state. In the latter, the
  first update returned successfully but the assertion's aggregate query
  failed before the repeated-update/idempotence check. Both are 06-X16;
  this does not establish that the unchecked path passes.

The exact package manifest, structured sweep log, leaf counts and
focused rerun logs are in session `files/pg-sweep-a3818782aa-*` and
`files/pg-sweep-unitstate-order-{postgresql,sqlite}.log`. These results
do not establish a full SQLite/dqlite run or a passed functional gate.

## PostgreSQL sweep 2

2026-11-05 at `08=ba634a0f13`, before the secret and unitstate X16 fixes
and the N17 key. Untagged `go test -count=1`, one package at a time, over
139 packages that use the database test harness (the N16 tiers A and B
plus test-only DB users). About 58 minutes serial. 102 packages pass; 37
fail. Session logs: `/tmp/pgval/sweep/` (temporary; failures only).

Classification by cause (failure counts are tests, not packages):

| Cause | Packages (failures) | Task |
|---|---|---|
| X16 production (42803) | secret (12), secret/state (50), unitstate/state (20), secretbackend/state (13), storage/state (part), removal model (1), application/state (2) | Secret and unitstate fixed in Stage 5 (`5e70fcbc84`, `0ed13ff885`, `3a8f0090c2`); secretbackend and storage by [05-X16-rest](05-queries.md#05-x16-rest-remaining-bare-grouped-columns-done) (`52a0eea6ee`, `640d3bcabf`). Removal and application are the test-side 06-X16 sites. |
| SQLite fixture DDL / triggers (42601) | changestream/stream (36), ssh/state/model (32), dbrepl (19), dbreplaccessor (4), changestreampruner (3), core/database/schema (1), machine/state (3), modelimport (9), modelimport/state/model (3), recovery/state (part), access/state (1) | 06-dialect-fixtures; modelimport and recovery deferred. |
| Bind types (42804, encode plans, 22P02) | crossmodelrelation/state/model (21), provisioner/state/model (5), blockdevice/state (6), controllerupgrader/state (4), resource/state (5), ssh/state/controller (1), storage/state (part) | 06-X25, 06-bind-types. |
| Missing function/operator (42883) | lease/state (4, `datetime`), access/state (1, `text = integer`), upgrade/state (1, U5) | 06-dialect-fixtures, 06-X29, 06-U5. |
| X27 time precision | status/state/model (49), macaroon/state (6), removal model (most of 47), operation (1), operation/state (part), application/state (part) | 06-X27. |
| Unknown column (42703) | operation/state (4), modelmigration/state/model (1), crossmodelrelation (1) | 06-unknown-column (triaged 2026-11-10: test SQL, X6 and X31). |
| Constraint mapping (23514, 23505) | network/state (5) | 06-constraint-errors (test-only, N20). |
| Error text, order, other | model/state/model (2, P0001 text), relation/state (1, order; X27 per the 2026-11-10 triage), domain root (2, `sqlite_schema`), controller/state root, lease/state | 06-errors, 06-order, 06-dialect-fixtures. |
| Not classified | upgradedatabase/upgradesteps (3), internal/migration (3, X21) | Import deferred; upgradesteps: 06-upgradesteps (X25, triaged 2026-11-10). |

Re-run after the Stage 5 fixes, 2026-11-05 at `05=3a8f0090c2`
(`domain/secret/...` and `domain/unitstate/...`, no 42803 left):

- `domain/secret/state`: 8 failures (was 50), all X27 precision of
  rotation and expiry times.
- `domain/secret`: 1 failure (was 12),
  `TestWatchSecretsRevisionExpiryChanges`: the two changes arrive in a
  different order (06-order).
- `domain/unitstate/state`: 4 failures (was 20). A new production defect,
  hidden behind X16 in sweep 2: `setUnitStateRelation` binds Go `int`
  relation IDs to the TEXT column `unit_state_relation.key`, which pgx
  refuses to encode. Stage 5 task 05-unitstate-relation-key.
- Service, modelmigration and root `unitstate` packages pass.

Re-run after 05-X16-rest and 05-unitstate-relation-key, 2026-11-05 at
`09=760f3b675f` (no 42803 left):

- `domain/unitstate/state` and `domain/removal/state/controller` pass.
- `domain/secretbackend/state`: 11 failures (was 13), all X27
  rotation-time `Equal` checks.
- `domain/storage/state`: 8 failures, all test-side: 2 `persistent` and
  2 `in_use` integer fixture binds (06-bind-types), 4 X27 time
  comparisons.

The later [PostgreSQL re-sweep](#postgresql-re-sweep) at
`09=a3818782aa` supersedes this run as the current baseline.

## D42 runtime-entry gate run

2026-10-26 at `08=c5f185e6f6` (05 `79f2e66049`, 06 `c52376b098`,
07 `e0c68da786`). Untagged `go test`, so the SQLite path is mattn SQLite,
not dqlite; Stage 9 still validates the real dqlite agent. Flags
`-race -count=1 -v`; packages run one at a time with a 1500 s cap.
Session logs: `files/d42/{sqlite,proof,pg}/`.

Selection (13 packages): `internal/database/postgresql`,
`internal/database/postgresql/testing`, `internal/database/drivererrors`,
`core/database/dialect`, `core/database/schema`, `internal/database`,
`domain/schema`, `generate/pgschemagen`, `agent/agentbootstrap`,
`domain/lease/state`, `domain/changestream/state`,
`domain/controllernode/state`, `core/watcher/eventsource`.
`domain/schema/...` expands to `domain/schema` plus three fixture packages
without tests. `internal/database/testing` has no tests; its consumers
exercise it.

- **SQLite:** all 13 ok; 246 pass, 0 fail, 47 skip (PG-only suites,
  17 `NodeManagerSuite` tests that need the dqlite tag, 5 upstream skips).
- **Backend proof:** with an unreachable PG URL, 10 packages fail; the
  backend-independent `drivererrors`, `dialect` and `pgschemagen` pass.
- **PostgreSQL:** all 13 rc=0, 0 failures. `TestPostgresBootstrapSuite`
  (2) and `TestFreshStateSuite` (5) pass.

| PG package | Pass | Skip | Skip reason |
|---|---:|---:|---|
| `domain/schema` | 7 | 93 | Blanket unapproved skip; 06-schema-skip. Accepted for this gate as a pass with a known skip gap (user decision, 2026-10-26). |
| `internal/database` | 18 | 17 | `NodeManagerSuite` needs the dqlite build tag. |
| `agent/agentbootstrap` | 8 | 1 | Upstream `TestBootstrapSuite/TestStub`. |
| `core/database/schema` | 22 | 0 | |
| `core/watcher/eventsource` | 26 | 0 | |
| `domain/changestream/state` | 7 | 0 | |
| `domain/controllernode/state` | 37 | 0 | |
| `domain/lease/state` | 12 | 0 | |
| `internal/database/postgresql` | 24 | 0 | |
| `internal/database/postgresql/testing` | 5 | 0 | |
| `generate/pgschemagen` | 10 | 0 | |
| `core/database/dialect` | 2 | 0 | |
| `internal/database/drivererrors` | 4 | 0 | |

Status triage: all 48 `domain/status/state/model` failures are X27 test
precision (06-status-diagnosis); none blocks runtime work. The full
`domain/removal/state/model` run has 47 failures, all mapped to recorded
test-side tasks (see the evidence table). Verdict: the runtime-entry gate
is met. The `domain/schema` skip gap and the test repairs remain open for
the functional gate.

## Latest recorded evidence

The [re-sweep](#postgresql-re-sweep) is the current full PostgreSQL
inventory; the rows below retain useful earlier per-package detail.
Undated rows are a snapshot from an earlier implementation handoff:
`06=eac393da16`, `05=d658b402a6`, `08=2e67edc999`.
The `restack/` run used that 06 tip.
See [history](../history/compatibility.md#historical-runs-and-restacking)
for logs. A missing command/build tag must be recovered before using a
historical pass to accept a new gate.

| Package | PG failures | Disposition |
|---|---:|---|
| `domain/lease/state` | 0 | 2026-11-09 at `09=a3818782aa`, full package with `-race`, no skips; claim and extend use database time. See [05-lease-db-time](05-queries.md#05-lease-db-time-lease-times-from-the-database-clock-done). |
| `internal/worker/leaseexpiry` | 0 | 2026-11-09 at `09=a3818782aa`, full package with `-race`, no skips; the real controller-schema expiry test passes (06-leaseexpiry-db). |
| `internal/changestream/stream` | 36 | 2026-11-09 at `09=a3818782aa`, N16 smoke with `-race`: every failure is the existing SQLite `AUTOINCREMENT` fixture rejected with 42601 (06-dialect-fixtures). Untagged SQLite passes. |
| `domain/objectstore/state` | 0 | Recorded pass. |
| `domain/keymanager/state` | 0 | 2026-10-26 at `08=7dc5b5bbc9`, full package; was 6 (X37, fixed). |
| `domain/storageprovisioning/state` | 0 | 2026-10-26 at `08=7dc5b5bbc9`, full package (659 s). |
| `domain/crossmodelrelation/state/model` | 23 | 2026-10-26 at `08=7dc5b5bbc9`. X37 offer-scope tests pass. Remaining, all test-side or U11: 16 fixtures bind integer `0` to `relation_status.updated_at` (mainly `package_test.go` `addOfferConnection`); 4 X25 `auto_prune` integer fixtures; 2 `instr` (X30, U11 secrets queries); 1 test `ORDER BY rowid` (`offer_test.go`, X31). After U11 was cherry-picked, `TestModelSecretsSuite` passes 29/29 at `08=c5f185e6f6`, so the 2 U11 failures are fixed. |
| `domain/crossmodelrelation` (U11 watcher tests) | 0 | 2026-10-26 at `08=c5f185e6f6`: `TestWatcherSuite/TestWatchRemoteConsumedSecretsChanges*` (2 tests). |
| `domain/secret/state` | 8 | 2026-11-05 at `05=3a8f0090c2`, full package: all X27 rotation and expiry time precision. Was 50 in sweep 2 (X16, fixed by 05-SECRET-X16). |
| `domain/secret` | 1 | 2026-11-05 at `05=3a8f0090c2`: `TestWatchSecretsRevisionExpiryChanges` receives two changes in the other order (06-order). Was 12. |
| `domain/unitstate/state` | 1 | 2026-11-10 at `09=a3818782aa`: unordered endpoint assertion (06-order), reproduced in both populated and unpopulated focused tests. Previously passed at `09=760f3b675f` after `1afd3b9434`; was 4 (relation key), 20 in sweep 2 (X16). |
| `domain/secretbackend/state` | 11 | 2026-11-05 at `09=760f3b675f`: all X27 rotation-time `Equal`. Was 13 (X16, fixed by `52a0eea6ee`). |
| `domain/removal/state/controller` | 0 | 2026-11-05 at `09=760f3b675f`, after `367f754dd6`. |
| `domain/storage/state` | 8 | 2026-11-05 at `09=760f3b675f`: 4 fixture binds (`persistent`, `in_use`; 06-bind-types), 4 X27. X16 fixed by `640d3bcabf`. |
| `domain/agentpassword/...` | 0 | 2026-11-05 after `ada2a852ee` and `283a00f630` (now `fa75871904`), both backends. |
| `internal/migration` | 3 | X21 (2), test trigger (1); PostgreSQL import deferred. |
| `domain/provisioner/state/model` | 5 | X25 fixtures. |
| `domain/provisioner` | 0 | Recorded pass. |
| `domain/modelmigration/state/controller` | 0 | Recorded pass; not proof of full migration support. |
| `domain/removal/state/model` | 47 | 2026-10-26 at `08=c5f185e6f6`, full package without `-race` (1333 s): 326 pass. All test-side, each mapped to a task above: 25 X27 schedule/time round trips; 11 06-X23 (`secret_test.go:299`); 2 06-X25 (`state_test.go` `force`); 2 06-DDL (`DATETIME` in `application_test.go`, `unit_test.go`); 2 06-X9 (`CONCAT`); 1 06-X16; 1 06-trigger-drop; 2 06-failure-injection; 1 06-errors (machine FK message). The older "22 X23" count was a miscount, and the 3 X38 failures are fixed. |
| `domain/status/state/model` | 48 | 2026-10-26 at `08=c5f185e6f6`, full package without `-race` (469 s): 125 pass, 48 fail, all X27 precision (see 06-status-diagnosis). |
| `domain/model/state/controller` | 0 | 2026-11-10 at `09=a3818782aa`. The earlier `TestStateSuite/TestGetActivatedModelUUIDs` order failure remains possible: its query (`WHERE uuid IN (...)`) has no `ORDER BY` (06-X17). |
| `domain/model/state/model` | 2 | 2026-10-27 at `06=e7f3822281`, full package with `-race`: 36 pass. `TestModelSuite/TestCreateModelAndUpdate` and `TestCreateModelAndDelete` match the immutable-row message exactly; PG prefixes `ERROR:` and suffixes the SQLSTATE (06-errors). |
| D42 runtime-entry packages (13) | 0 | 2026-10-26 at `08=c5f185e6f6`; see [D42 runtime-entry gate run](#d42-runtime-entry-gate-run). `domain/schema` has a known skip gap (06-schema-skip). `TestPostgresBootstrapSuite` passes, superseding Stage 8's old operation-4 failure. |

The [re-sweep](#postgresql-re-sweep) supersedes older package outcomes
above. `domain/modelimport` execution stays deferred for the functional
gate.

A package's presence in this evidence inventory does not automatically
make it a prerequisite for every milestone. Apply the D42 entry list above;
record other packages' disposition in the functional gate when its exact
workload is selected.
