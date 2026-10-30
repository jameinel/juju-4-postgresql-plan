# Stock upstream Juju 4.1 changes

This is a standalone implementation plan for a stock upstream 4.1
checkout. Read this guide's shared rules, the assigned task, and the target
repository's root/nested AGENTS.md and contribution guidance. No other
planning document, checkout, configuration or test setup is required.
Describe each fix in terms of upstream behaviour and its regression tests.

## Working agreement

- Refresh and record the upstream 4.1 base (`B`). Check current code and
  existing reviews for completed work before implementing anything.
- Start each independent branch from `B`. The integer-schema work on
  `4.1-sqlite-integers` is merged; do not duplicate it or add unrelated
  work to that branch. For a separately approved follow-up, branch from a
  refreshed upstream `4.1` and preserve the merged branch's history.
- Dedicated worktrees are allowed. Assign one branch/worktree per owner;
  never switch, reset or rebase another owner's checkout. A worktree's
  presence does not imply permission to modify it.
- When creating the worktree add a _deps symlink back to ../juju/_deps so
  that the dependencies can be shared rather than redownloaded in the new tree.
- Keep one review concern per branch. Multiple commits are fine when they
  serve that concern. Shared generated files are an integration conflict,
  not automatically a reason to stack independent reviews.
- Implement using the APIs, schema layout, helpers and test infrastructure
  present in the target branch. Missing helpers require a small local
  solution or clarification, not dependencies on another project.
- Stop for unresolved behaviour, unexpected dependencies or ownership
  conflicts. Do not weaken tests to make a patch pass.

## Existing integer branch

PR #23680 on `4.1-sqlite-integers` merged to upstream `4.1` on
2026-10-30. Its head `0e9be8ddd7` includes the lookup-ID and relation-status
integer corrections, revision-updater unit-count fix, and review follow-up
(full-row conversion coverage, invalid-ID classification, explicit view row
counts and schema-evolution documentation). Do not recreate or duplicate
these changes. The review evidence is in
[COMMENT-4.1-sqlite-integers-review.md](COMMENT-4.1-sqlite-integers-review.md).

## Candidate branch map

Rows with a PR have an established branch; other names are proposals. Each
row defines an upstream problem to solve, not a patch to copy.

| Branch | Change and entry points | Required result |
|---|---|---|
| `4.1-sqlite-integers` (**landed**, PR #23680) | `relation_status_type.id` and `relation_status.relation_status_type_id`: TEXT declarations disagree with integer writers and lookup values. | INT declarations, matching export types and legacy string-ID conversion; preserve the existing unit-count and lookup-ID fixes. |
| `4.1-binary-key-columns` (**landed**, PR #23682) | `user_activation_key.activation_key`, eight `bakery_config` key columns and `macaroon_root_key.root_key`: raw bytes declared TEXT. | BLOB declarations and consistent storage/export/recovery representations, including legacy compatibility. |
| `4.1-removal-force-boolean` (**landed**, PR #23690) | `removal.force`: INT declaration despite boolean writers/readers. | BOOLEAN declaration, boolean export type and validated 4.0.12 integer conversion. |
| `4.1-model-config-precedence` (**in review**, PR #23683) | `v_model_config` and `domain/modelconfig/state`: duplicate agent stream/version rows let row ordering choose the winner. | Authoritative `agent_version` values win regardless of config contents or row order. |
| `4.1-keymanager-insert-id` (**landed**, PR #23684) | `domain/keymanager/state`, `ensureUserPublicKey`: a zero-row insert can leave `LastInsertId` referring to an earlier key. | Use the inserted row's returned ID; reject unknown algorithms without linking an unrelated key. |
| `4.1-controller-node-id-text` (**landed**, PR #23685) | `internal/database/bootstrap.go`, `InsertControllerNodeID`, and changestream/upgrade fixtures: uint64 values bound to a TEXT column. | Bind decimal strings and support the full unsigned range. |
| `4.1-lease-migration-tests` (**landed**, PR #23686) | `domain/lease/state/migration_test.go`: ignored seed errors can invalidate test setup. | Assert setup success and genuinely test held/expired leases with appropriate timestamp fixtures. |
| `4.1-crossmodel-secret-revisions` (**in review**, PR #23688) | `domain/crossmodelrelation/state/model/secrets.go`: grouped queries select an arbitrary consuming unit's revision. | Report a secret if any remote consuming unit is behind and return its latest revision once. Join through the indexed offer connection, with matching schema, migration and removal changes. Local-consumer query portability is split into `4.1-local-secret-revisions`; see the recorded scope below. |
| `4.1-dbaccessor-pool-cleanup` | `internal/worker/dbaccessor/tracker.go`: pools opened during setup or reopen attempts are leaked when a later step fails. | Close every pool the tracker opens but does not retain, including failures setting foreign keys, model initialization and reopen ping; preserve pools successfully transferred to the worker. |
| `4.1-nullable-primary-keys` (**in review**, charm-only PR #23710) | `charm_container_mount` and `charm_manifest_base` have nullable composite-primary-key columns. | Require the container key; key manifest bases by manifest position, with an import diagnostic/test for duplicate positions. Published head `ba21f48ad3` on upstream `8369e6e2a9`. |
| `4.1-application-exposure-scope` (**in review**, PR #23723, stacked on charm) | `application_exposed_endpoint_cidr` / `_space` use NULL `application_endpoint_uuid` for wildcard exposure; only these exposure tables store NULL. | Move all-endpoint exposure to application-wide `application_exposed_space` / `_cidr`, mirroring bindings; require endpoint IDs in endpoint tables. Preserve readers/API/`""` wildcard behaviour, split legacy import by scope, collapse duplicate wildcard rows, and avoid `remove-space --force` UNIQUE errors. Includes the combined nullable-primary-key guard. Published head `f31de64e52`; land charm first and rebase before landing exposure. |
| `4.1-agentbinary-conflict` | `RegisterAgentBinary` in `domain/agentbinary/state/{controller,model}/state.go`: queries after a primary-key conflict inside the same transaction. | Avoid the conflict in SQL (`ON CONFLICT DO NOTHING`, then compare), keeping `AlreadyExists` / `AgentBinaryImmutable`. Test new, repeated and different-object registrations. |
| `4.1-lease-pin-conflict` | `domain/lease/state`, `PinLease`: ignoring a unique violation leaves the transaction unusable on PostgreSQL. | Resolve the pin race in SQL with `ON CONFLICT DO NOTHING`; preserve the existing result for already-pinned leases and test competing pins. |

### Schema lane

The integer-schema corrections have landed. Nullable primary keys have a
published two-PR split: charm keys in PR #23710 and stacked exposure in
PR #23723. The binary-key, removal-boolean, keymanager, node-ID
and lease-test changes have landed; do not recreate them.

For schema follow-ups, inspect schema SQL under
`domain/schema/{model,controller}/`, generated export types under
`domain/export/types/`, and legacy conversion code under
`domain/modelimport/transformer/transforms/to_v4_1_0/`. Assign shared
handwritten changes explicitly.

Regenerate from each branch's own schema using its export/import/recovery
tooling (currently `make rebuild-export`). After a merge conflict,
regenerate from the combined schema rather than selecting one branch's
entire generated file. Confirm the target release's schema-change policy:
edit beta-only declarations where permitted, otherwise use its incremental
patch mechanism and test supported upgrades.

Required checks:

- Integer IDs: valid, nullable where supported, and invalid legacy string
  representations through transform/import/export and persistence.

### Leaf task requirements

- **Keymanager:** test existing/new keys, unknown algorithms after a prior
  insert and unchanged associations. Keep general SQL quotation/equality
  cleanup out of this fix.
- **Node IDs:** exercise high-bit and max-uint64 values through the actual
  production insert/read path as decimal text. Include matching fixture
  fixes; do not change the column to signed INT.
- **Lease tests:** use the target branch's time seams/helpers or a small
  test-local fixture. Preserve held/expired and boundary semantics without
  changing production lease timing as part of this test-only task.
- **Model config:** exclude authoritative keys from the competing config
  rows. Test values that sort before and after the authoritative values,
  and assert one effective value for each key.
- **Dbaccessor pools:** inject failures after opening a pool at the foreign
  keys pragma, model initialization and reopen ping steps. Assert every
  unretained pool is closed and successfully retained pools remain usable.
- **Lease pin conflicts:** exercise competing pins and confirm the losing
  operation leaves the transaction usable and preserves the existing
  already-pinned result.
- **Cross-model secrets:** implement the current review contract: report a
  secret if any remote consuming unit is behind, return its latest revision
  once, and match consumers through their indexed offer connection.
  Cover no consumers, current/behind consumers, mixed multi-unit revisions,
  multiple secrets/revisions, application-name prefix collisions and
  optional filtering. Assert membership, chosen revisions and duplicates;
  a single-unit test does not establish the contract.

## Current work queue

The landed branches in the map need no further implementation. GitHub is
the source of truth for current review and CI status. U15 is published as
charm-only juju/juju#23710 and stacked exposure juju/juju#23723, alongside
open reviews #23683, #23688, #23691, #23693, #23694 and #23720. The review's
manifest-position diagnostic/tests and table-specific unset errors are
implemented and published in their respective PRs. Keep work on existing
review branches rather than recreating them. New independent implementation tasks are dbaccessor pool
cleanup, agent-binary conflict handling and lease pin conflict handling. The
dbrepl latest-row query rewrite is queued separately as
`4.1-dbrepl-latest-row`, following the review request on PR #23691; it is not
covered by that PR's changestream-only rewrite. Secret latest-revision work
follows the cross-model secret review, and lookup-key `NOT NULL` work follows
an independent `4.1-lookup-key-not-null` session; it does not need either
U15 branch to land first.

## Nullable primary-key review split

The 2026-10-30 review of juju/juju#23710 requests separate charm-key and
application-exposure PRs. The approved local split keeps the existing
`4.1-nullable-primary-keys` branch/PR for charm keys and places exposure on
`4.1-application-exposure-scope`, stacked on the charm branch. This is an
explicit exception to the independent-base agreement: the schema-wide
primary-key guard passes only with both concerns fixed and belongs to the
second PR. No unrelated PoC work is included.

- Charm PR: `0015-charm.sql`, charm schema regressions, mount export type,
  legacy mount and manifest-position deltas, generator support for a
  constraint-only delta, matching generated transforms/recovery and tests.
  Duplicate manifest positions halt with charm UUID and both indices rather
  than a raw UNIQUE error; nullable attributes remain valid.
- Exposure PR: `0019-application.sql`, four-table exposure writes/view,
  watcher namespaces/triggers, export/import/recovery, application removal,
  space checks/reset-to-alpha, their tests and the combined primary-key
  guard. Deletion errors identify the failed SQL/table; failed deletions
  roll back preceding tables in the same transaction.
- Each branch was regenerated from its own schema; the exposure tree was
  regenerated after restacking on the charm follow-up, retaining both
  concerns' deltas rather than choosing one whole generated file.
- Land charm first. Until then, review exposure against the charm head;
  `juju/juju` cannot target a fork-only base branch, so the proposed PR to
  `4.1` discloses its dependency and provides a scoped comparison. The
  2026-10-30 follow-up refreshed both branches onto upstream `8369e6e2a9`,
  resolved conflicts and regenerated the stack. After charm lands, rebase
  exposure onto `4.1` before landing it.
- `4.1-lookup-key-not-null` is an independent follow-up with its own
  INT-key-scoped schema guard; it does not depend on the exposure guard.
  Coordinate global-guard tightening after integration, as described below.
  Both U15 fixes are required for PoC reintegration; charm alone does not
  repair the bootstrap's wildcard-exposure failure.

Current local revisions: base `8369e6e2a9`; charm head `ba21f48ad3`
(`7001a1b74b` + diagnostic/generator follow-up); exposure head `f31de64e52`
(`4d9518117c` + deletion diagnostic). The pre-refresh split was based on
`9ea82c03bd`, charm `4df04695d8` and exposure `3c90a7ded3`; those heads are
preserved in `backup/charm-before-base-refresh-20261030` and
`backup/exposure-before-base-refresh-20261030`. After explicit approval,
both branch heads and exact PR descriptions were published: charm-only
[juju/juju#23710](https://github.com/juju/juju/pull/23710), stacked exposure
[juju/juju#23723](https://github.com/juju/juju/pull/23723), and the
[reviewer-context-only reply](https://github.com/juju/juju/pull/23710#issuecomment-6527752522).
Published content is recorded in [the charm PR](PR-4.1-nullable-primary-keys.md),
[the exposure PR](PR-4.1-application-exposure-scope.md) and
[the reply](COMMENT-4.1-nullable-primary-keys-review.md). No threads were
resolved. New-head CI has not been assessed. Original combined work is
preserved in `backup/nullable-primary-keys-before-split-20261030`.

Initial split validation (pre-refresh revisions): charm schema, legacy import/delta and
recovery-state race suites passed; the generator suite passed after fixing
a test helper typo. Exposure schema, application integration/service,
network-state, removal-model-state, import and recovery-state race suites
passed before the restack. After restacking, all generator/import/recovery
packages and the focused exposure-state race suite passed. A broad
application-state suite hit its cumulative 8-minute timeout during
`TestUnitStateSuite/TestGetUnitWorkloadVersion`; it is not a full green run.
Charm-import stress reported 12 completed runs and exposure-watcher stress
6, both with zero failures before the intentional outer timeout. Both
`make pre-check` invocations passed but skipped static analysis; explicit
focused `golangci-lint` runs reported zero issues. IDE build validation was
unavailable for these unopened worktrees; the Go race suites compiled the
changed packages. The final stacked tree also passed
`timeout 600s make go-build GOFLAGS=-p=2` for all linux/amd64 agent/client
targets using existing `_deps`. Live-controller QA and PostgreSQL tests
were not run.
Refreshed-head validation: all nine charm generator/schema/import/recovery
race packages passed; all 13 exposure generator/schema/application/service/
import/recovery/network/removal race packages and the focused exposure-state
suite passed. Regeneration is idempotent. Charm-import stress reported 132
completed runs and exposure-watcher stress 6, both with zero failures before
intentional outer timeouts. Explicit build-tagged focused lint reported
zero issues. The refreshed base and final stack both passed `make go-build`
for all linux/amd64 targets. The full application-state suite and live QA
were not rerun. Generated trigger-template trailing whitespace remains;
scoped handwritten diff checks pass. Exact commands and provenance are in
[UPSTREAM-FIXES.md](UPSTREAM-FIXES.md#nullable-primary-key-base-refresh-2026-10-30).

## Cross-model secret review scope and follow-up

The review follow-up for `juju/juju#23688` keeps the remote any-unit-behind
query fix, indexed offer-connection schema/export/import/recovery changes,
consumer cleanup, and their regression tests together.

The independent local-consumer query portability change is split out onto
`4.1-local-secret-revisions`, based on `9ea82c03bd`, and published as
[PR #23720](https://github.com/juju/juju/pull/23720). It owns only
`domain/secret/state/state.go` methods
`InitialWatchStatementForConsumedSecretsChange` and
`GetConsumedSecretURIsWithChanges`, and
`TestConsumedSecretsChangesMultipleUnitsAndRevisions` in `state_test.go`.
The regression passes with both the original SQLite SQL and the portable
SQL; the full secret-state race suite and focused lint passed locally. The
schema-dependent remote-consumer fixture stays in `juju/juju#23688`.
The cross-model review follow-up is committed as `29fc497b6e`; the extracted
local-query change is committed as `369fb64c2d`. PR #23720 remains open; its
review suggests explicitly testing the empty-consumer result from both
local-query methods; those cases are not yet present in the local branch. See
GitHub for current CI status.

Do not confuse this extracted change with `4.1-secret-latest-revision`
below: that task covers five other latest-revision queries and remains
unimplemented. Per-connection watcher scoping is also separate follow-up;
`juju/juju#23688` still scopes its watcher by the offered application.

Explicit migration policy for `juju/juju#23688`: warn and discard
unresolvable
remote-consumer revision-tracking rows, including malformed names. These
rows memoise consumed revisions, not secret content, grants or core model
structure. A normal remote-secret read recreates missing tracking at the
latest revision; dropping it does not itself emit a watcher event. Valid
tracking is preserved. This is a justified exception for reconstructible
metadata, not a general policy permitting inconsistent model data to be
dropped.

## Follow-on SQL improvements

Assign these separately after confirming the upstream benefit and behaviour
contract. Portability is a valid rationale, but not permission to combine
unrelated changes or label observable behaviour changes as mechanical.

| Proposed branch | Change / entry point | Review requirement |
|---|---|---|
| `4.1-changestream-latest-row` (**in review**, PR #23691) | `internal/changestream/stream/stream.go`: replace bare-column `MAX(id)` grouping with explicit latest-ID selection and a join to its row. | Preserve full-row consistency, high-water marks, coalescing and pruning; add regressions and required concurrency coverage. The equivalent dbrepl query is tracked separately below. |
| `4.1-dbrepl-latest-row` (**queued**, follow-up to PR #23691) | `internal/worker/dbrepl/worker.go` (review location: line 680): replace `SELECT MAX(c.id), t.edit_type, n.namespace, changed, created_at ... GROUP BY c.namespace_id, c.changed` with a CTE selecting the latest ID per namespace/changed key, then join back to that change-log row. Join `change_log_edit_type` in the outer select to retain `t.edit_type`, as well as the namespace lookup. | Keep all returned fields from the selected latest row and preserve coalescing, query bounds, ordering and replication progress. Add regressions in `internal/worker/dbrepl/worker_test.go` for repeated changes to one key with differing edit types/timestamps, multiple keys/namespaces, empty results and query-boundary cases; verify the replicated payload and progress, not only the SQL result. Behaviour-preserving tests must pass with both old and rewritten SQL. Run the affected package tests with race/stress coverage. Confirm the query is still outstanding before dispatch; use a separate branch/worktree, not the PR #23691 owner's checkout. |
| `4.1-blockcommand-order` (**landed**, PR #23692) | `domain/blockcommand/state`: replace implicit `rowid` ordering with explicit block-type order. | Preserve and test the agreed explicit order. |
| `4.1-operation-query-portability` (**in review**, PR #23693) | `domain/operation/state`: boolean parameter comparisons and text-based size expressions. | Separate commits; preserve filtering and pruning thresholds, including value-type coverage. |
| `4.1-objectstore-conflicts` (**in review**, PR #23694; sentinel follow-up published) | `domain/objectstore/state`: use one conflict clause instead of separate clauses for each hash. | Return the documented `objectstoreerrors.ErrMetadataUUIDAlreadyExists` sentinel when a UUID exists with different hashes or size, distinct from `ErrHashAndSizeAlreadyExists`. Test hash and size-only UUID conflicts, duplicate UUID reuse, unchanged metadata/paths/hints, and wrapped sentinel propagation through both service put methods. |
| `4.1-sql-test-literals` (**landed**, PR #23696) | SQL test helpers and production SQL: single-quoted strings instead of SQLite's double-quoted-string fallback. | Bound the file set; preserve fixture and query meaning and coordinate overlapping owners. |
| `4.1-local-secret-revisions` (**in review**, PR #23720) | `domain/secret/state/state.go`: make `InitialWatchStatementForConsumedSecretsChange` and `GetConsumedSecretURIsWithChanges` portable by selecting the latest revision explicitly. | Preserve results across multiple units/revisions and optional filters; add explicit empty-consumer cases for both methods (reviewer suggestion, not yet present in the local branch). |
| `4.1-secret-latest-revision` | `domain/secret/state/state.go`: `updateSecret`, `GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and `getSecretsRevisionExpiryChanges` read ungrouped columns beside `MAX(revision)` and rely on SQLite returning them from the latest-revision row. Select the latest revision explicitly instead. This follows the consumed-secret changes in PRs #23688 and #23720. | Show the results are unchanged with several revisions, expiry and rotation rows, and both owner kinds. Tests must pass against the old and the new queries. |
| `4.1-sql-standard-syntax` (**landed**, PR #23695) | Two commits. Equality: replace SQLite's `==` with `=` in `domain/keymanager/state` `DeletePublicKeysForUser`, `domain/storageprovisioning/state` `GetVolumeAttachmentIDs` and `domain/crossmodelrelation/state/model` `ValidateApplicationAndEndpointsForOffer`. Join scope: `domain/removal/state/model` `GetDetachInfoForStorageAttachment` joins a one-row aggregate CTE with a comma and then uses `JOIN ... ON` with the table before the comma; spell that join as `CROSS JOIN`. | No behaviour change. Test that removing a user's keys deletes keys no other model uses and keeps shared ones. Test detach info when the requested attachment is no longer alive and no other attachment is, so the fulfilment count is zero. Confirm no other `==` remains in production SQL. |
| `4.1-lookup-key-not-null` (**planned, independent; separate session**) | 65 single-column `id INT PRIMARY KEY` tables allow NULL: 48 model and 17 controller at upstream `8369e6e2a9`. Export generation maps them to `ID *int64`. SQLite's INT is not the rowid-aliasing INTEGER type. | Declare `id INT PRIMARY KEY NOT NULL`, regenerate to `ID int64`, validate legacy IDs without inventing zero, and preserve seed/import/recovery behaviour. Start from refreshed upstream `4.1`, not either U15 branch. Add an INT-key-scoped guard that does not require unrelated composite keys to be fixed. See the self-contained task below. |

## Lookup-key NOT NULL task (separate session)

Approved on 2026-10-30 as an **independent** review concern. Do not implement
it during the charm/exposure review session. Start `4.1-lookup-key-not-null`
in its own worktree from freshly fetched upstream `4.1`; the inspected base
was `8369e6e2a9` (includes the merged integer-schema PR #23680).

### Contract and bounded scope

- Find every single-column primary key whose declared type is exactly
  `INT` and whose `pragma_table_info` notnull flag is zero. At the inspected
  base this is 48 model tables and 17 controller tables, all named `id`.
  Examples: model `agent_stream`, `life`, `os`, charm lookups and status
  lookups; controller `cloud_type`, `lease_type`, permission lookups and
  `upgrade_state_type`. Both schemas include `change_log_edit_type` and
  `change_log_namespace`. Re-inventory at the session's actual base.
- Change these declarations in `domain/schema/{model,controller}/sql/`
  to `id INT PRIMARY KEY NOT NULL`. Leave INTEGER rowid keys, composite
  keys, historical release DDL and 4.0.12 export types unchanged. Confirm
  the then-current beta/schema-upgrade policy before editing base patches.
- Run `make rebuild-export`. `generate/export/main.go` already maps INT to
  int64 and adds a pointer only when NOT NULL is absent, so no blanket
  primary-key/type-generator override is needed. Regenerate export types,
  import/transform and recovery from this branch's own schema. Only these
  65 key fields should lose their pointers, alongside required generated
  consumers; preserve unrelated nullable attributes.
- Ordinary seed rows and `domain/cloud/state.AllowCloudType` already write
  integer IDs. Preserve their IDs and seeded-import `ON CONFLICT DO NOTHING`
  behaviour. Zero is a valid lookup ID, not evidence that an ID is missing.
- For imported 4.0.12 lookup rows, add typed deltas (sharing a small helper
  if useful) that preserve every column and reject nil IDs with table/row
  context and the appropriate invalid-payload classification. A populated
  pointer containing zero must succeed. Struct-shape changes automatically
  trigger delta generation: no dependency on the charm constraint-only
  generator enhancement exists.
- Respect `generate/modelimport/main.go`'s target-local exclusions.
  `change_log_edit_type` and `change_log_namespace` are exported but are
  not imported from the source. Preserve their existing exclusion contract;
  do not reintroduce their rows merely because their generated types change.
- Controller types exist only at 4.1.0 and require no legacy controller
  delta. Model/controller 4.1.0 YAML decoding in `domain/export/payload.go`
  currently uses `yaml.Unmarshal`, which can turn explicit NULL or omitted
  scalar fields into zero. Validate presence/nullability of the changed IDs
  before losing that distinction, where those rows are consumed. Do not
  claim an int64 field or SQL NOT NULL alone rejects malformed wire IDs,
  and do not broaden this into validation of every existing scalar field.
- Recovery's `domain/recovery/service/decode.go` already checks raw column
  presence and rejects NULL for non-pointer scalars. Verify this through
  generated model/controller dump decode and replay, rather than assuming
  generation establishes the new contract.

### Independence and integration

This task changes lookup keys, not charm/exposure keys. Shared SQL files
(such as `0015-charm.sql`), generated export/transform/recovery files and
changestream namespaces are merge conflicts, not semantic prerequisites.
The independent branch owns a new `lookup_primary_key_schema_test.go`
(or equivalent existing scoped suite) asserting NOT NULL for single-column
INT primary keys in both schemas. It must pass on upstream even while the
four U15 composite-key tables remain unfixed; do not import the schema-wide
U15 guard and introduce a hidden dependency.

If the exposure guard has landed by the session's base, tighten it to exempt
only single-column INTEGER keys as part of this concern. Otherwise retain
its current INT exemption on the exposure branch, and reconcile it when
lookup and exposure changes meet: regenerate the combined schema, remove
the INT exemption, and run both scoped/global guards. Do not narrow the
global guard before the lookup declarations land, or make either independent
PR include the other's schema fix. Keep this integration obligation explicit
in the future PR description/handoff.

### Required checks and stop conditions

- Test fresh model/controller schemas reject NULL INT keys and retain valid
  seed IDs. Assert the count/set of affected declarations and generated
  non-pointer fields; don't just test one lookup table.
- Exercise each consumed legacy lookup type's complete row through its
  delta, including nil versus populated zero, multiple rows and non-ID
  nullable attributes. Use representative real legacy decode -> transform
  -> import -> export round trips for model persistence, including duplicate
  existing seed rows and a valid non-seed row where permitted.
- Test 4.1.0 model/controller raw YAML with valid zero/non-zero, explicit
  NULL and omitted IDs at the actual decoder boundary; excluded source
  changestream rows must stay excluded. Typed Go rows cannot represent the
  difference between missing and zero after decoding.
- Test model/controller recovery dump round trips, explicit NULL/missing
  ID rejection and no partial replay on failure. Do not alter immutable-table
  replay/bootstrap requirements to accommodate fixtures.
- Run race-enabled schema/export/import/recovery and relevant cloud-state
  suites; stress suites using goroutines per repository guidance. Run gci,
  focused lint, build and regeneration-idempotence checks. After conflicts,
  regenerate from the combined schema and rerun the boundary tests.
- Stop for changed release policy, newly discovered nullable-key writers,
  or a representation that cannot be preserved. Record base, inventory,
  commits, validation and guard-integration status. No PostgreSQL work or
  unrelated composite-key cleanup belongs in this branch.

Further candidates need named files and tests before dispatch: explicit
upsert targets/qualification, explicit CTE membership subqueries and
`CASE` in model connection queries.
Conflict-handling changes must be split by operation and preserve
idempotency, retries and errors. Do not mix a production time-calculation
or aggregation rewrite into a syntax-only cleanup.

## Objectstore review follow-up and process

The 2026-10-29 follow-up for `juju/juju#23694` is published as
`53feca2259` on the existing `4.1-objectstore-conflicts` branch. Its parent
is `87ac64185e`; the branch base is `04bc64ef54`. Upstream `4.1` was checked
at `7ce6f49c04`; this review fix does not rebase or modify another owner's
checkout. The approved PR description is published, together with the
[review reply](https://github.com/juju/juju/pull/23694#issuecomment-6510378577).
- The explicit SQL table/projection-alias request is already implemented
  and its review thread is resolved. Keep that fix.
- The remaining suggestion is addressed by the documented UUID-conflict
  sentinel, preserving the error text and hash-conflict classification.
- Dqlite race suites for `domain/objectstore/...` pass. Focused metadata
  stress reports 10 runs and no failures before the intentional timeout.
  `make pre-check` passes but skips static analysis by default; the explicit
  `golangci-lint run ./domain/objectstore/...` reports zero issues.
- The sentinel regression was reproduced before changing the state return:
  both existing different-hash UUID tests failed `ErrorIs` against the bare
  error. Size-only UUID and both service-wrapper regressions pass with the
  fix. No deployment smoke test or PostgreSQL test was run. The IDE build
  tool cannot target this unopened worktree; the Go race suites compile it.

Validation commands, run from the objectstore worktree (the session-local
stress binary was removed afterwards):

```sh
make run-go-tests TEST_PACKAGES="./domain/objectstore/state" \
  TEST_ARGS="-race -count=1 -p 2" \
  TEST_FILTER="TestStateSuite/TestPutMetadata.*SameUUIDDifferentHashes"
# Before the state fix: both selected tests fail ErrorIs as expected.
make run-go-tests TEST_PACKAGES="./domain/objectstore/..." \
  TEST_ARGS="-race -count=1 -p 2"
binary=/home/jameinel/.copilot/session-state/8276f750-86ce-4836-9ab6-f4e68a0cb20f/files/objectstore-state.test
make run-go-tests TEST_PACKAGES="./domain/objectstore/state" \
  TEST_ARGS="-race -p 2 -c -o $binary"
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run 'TestStateSuite/TestPutMetadata'
# Exit 124 is the expected outer timeout; 10 runs, zero failures reported.
make pre-check
golangci-lint run ./domain/objectstore/...
git --no-pager diff --check
```

The Makefile supplied `-mod=readonly -tags=dqlite,libsqlite3`, CGO enabled,
its existing musl/Dqlite dependencies, and external static linking. Go
version: `go1.27.3`. `gofmt` and the prescribed three-section `gci write`
were run on all four changed Go files. No dependencies were installed.

Process steps for this review:

1. Completed: reviewed the local commit, revised
   [PR description](PR-4.1-objectstore-conflicts.md) and
   [review reply](COMMENT-4.1-objectstore-conflicts-review.md).
2. Completed: received explicit approval to push the commit to
   `jameinel/juju:4.1-objectstore-conflicts`, replace the description of
   `juju/juju#23694`, and post the exact reviewed reply there.
3. Completed: published only those approved updates. The installed
   `gh pr edit` failed on deprecated classic-project GraphQL fields; the
   approved description was published using
   `gh api --method PATCH repos/juju/juju/pulls/23694` instead. No review
   threads were changed; further published edits/resolutions need approval.
4. After the PR lands, reconcile the PoC's UUID-reuse classification with
   the upstream sentinel; do not treat UUID reuse as content already stored.
   That integration work is separate from this review fix.

## Separate agent-workflow documentation task

`4.1-pr-description-guidance`: clarify PR-description versus review-comment
content in the agent commit instructions. Separate from the SQL fixes;
[patch](4.1-pr-description-guidance.patch) and
[PR draft](PR-4.1-pr-description-guidance.md) staged here. No branch or PR yet.

## Dispatch

Supply the upstream base, one named branch/worktree, allowed paths,
expected behaviour, prerequisites and required regression cases. The
executor reads this guide and the target repository's instructions; task
selection does not authorise work on other branches.

Example assignment:

> In stock upstream Juju 4.1 at <base>, implement the dbaccessor pool-cleanup
> task from UPSTREAM-PLAN.md on <branch> in <worktree>. Use only that
> checkout's code and test infrastructure. Add the specified regressions,
> run its normal validation, and report the diff and results. Stop for
> unapproved semantics or dependencies.

## Upstream validation and review

- Reproduce the upstream defect where applicable, then add the regression
  alongside the fix. For a behaviour-preserving SQL improvement, assert the
  existing contract across relevant edge cases.
- Run focused tests first, then the affected package suites. Starting
  scopes: `domain/keymanager/state`; `internal/database` plus
  `domain/changestream/state` and `domain/upgrade/state`;
  `internal/worker/dbaccessor`; `domain/lease/state`;
  `domain/modelconfig/state` plus schema tests; and
  `domain/crossmodelrelation/state/model` plus affected consumers.
- Use the target checkout's standard test commands, dependencies and
  database fixtures. Include actual dqlite checks where required, and its
  prescribed race/stress, watcher-readiness, formatting and lint checks.
- For schema changes, exercise writers -> storage -> export -> legacy
  import -> consumers. Cover supported representations in both directions,
  relevant recovery paths and the release's supported upgrade mechanism.
  Generation alone is not acceptance; generated output must be limited to
  the intended changes.
- Record the base, final commit, exact commands/build tags and results.
  State tests not run and unresolved risks; do not claim broader coverage.
  Explain the upstream defect/benefit and any behaviour change in the PR,
  following the target repository's template.

Return the branch, commits, test evidence and review link to the coordinator.
The coordinator updates shared planning records; parallel agents need not
edit them. Respect the host's test resource budget and other owners' state.
Each independent review targets upstream 4.1; rebase/regenerate when needed
after related changes land, without combining unrelated reviews.
