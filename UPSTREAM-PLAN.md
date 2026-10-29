# Stock upstream Juju 4.1 changes

This is a standalone implementation plan for a stock upstream 4.1
checkout. Read this guide's shared rules, the assigned task, and the target
repository's root/nested AGENTS.md and contribution guidance. No other
planning document, checkout, configuration or test setup is required.
Describe each fix in terms of upstream behaviour and its regression tests.

## Working agreement

- Refresh and record the upstream 4.1 base (`B`). Check current code and
  existing reviews for completed work before implementing anything.
- Start each independent branch from `B`. Use the existing
  `4.1-sqlite-integers` branch for its assigned extension, preserving its
  current commits and review history; coordinate changes with its owner.
- Dedicated worktrees are allowed. Assign one branch/worktree per owner;
  never switch, reset or rebase another owner's checkout. A worktree's
  presence does not imply permission to modify it.
- Keep one review concern per branch. Multiple commits are fine when they
  serve that concern. Shared generated files are an integration conflict,
  not automatically a reason to stack independent reviews.
- Implement using the APIs, schema layout, helpers and test infrastructure
  present in the target branch. Missing helpers require a small local
  solution or clarification, not dependencies on another project.
- Stop for unresolved behaviour, unexpected dependencies or ownership
  conflicts. Do not weaken tests to make a patch pass.

## Existing integer branch

PR #23680 on `4.1-sqlite-integers` contains the lookup-ID corrections and
revision-updater unit-count fix. Check its current state before changing it;
do not duplicate those fixes. The relation-status integer correction is a
separate commit on that branch, reusing its legacy-ID parsing helpers and
extending its import fixtures.

The 2026-10-29 review follow-up adds full-row conversion coverage, invalid-ID
error classification, explicit view row counts and developer schema-evolution
documentation. Verified and committed as `0e9be8ddd7`; awaiting publication.
Details and evidence are in
[COMMENT-4.1-sqlite-integers-review.md](COMMENT-4.1-sqlite-integers-review.md).

## Candidate branch map

Rows with a PR have an established branch; other names are proposals. Each
row defines an upstream problem to solve, not a patch to copy.

| Branch | Change and entry points | Required result |
|---|---|---|
| `4.1-sqlite-integers` (existing; PR #23680 open) | `relation_status_type.id` and `relation_status.relation_status_type_id`: TEXT declarations disagree with integer writers and lookup values. | INT declarations, matching export types and legacy string-ID conversion; preserve the existing unit-count and lookup-ID fixes. |
| `4.1-binary-key-columns` (**landed**, PR #23682) | `user_activation_key.activation_key`, eight `bakery_config` key columns and `macaroon_root_key.root_key`: raw bytes declared TEXT. | BLOB declarations and consistent storage/export/recovery representations, including legacy compatibility. |
| `4.1-removal-force-boolean` (**landed**, PR #23690) | `removal.force`: INT declaration despite boolean writers/readers. | BOOLEAN declaration, boolean export type and validated 4.0.12 integer conversion. |
| `4.1-model-config-precedence` (**in review**, PR #23683) | `v_model_config` and `domain/modelconfig/state`: duplicate agent stream/version rows let row ordering choose the winner. | Authoritative `agent_version` values win regardless of config contents or row order. |
| `4.1-keymanager-insert-id` (**landed**, PR #23684) | `domain/keymanager/state`, `ensureUserPublicKey`: a zero-row insert can leave `LastInsertId` referring to an earlier key. | Use the inserted row's returned ID; reject unknown algorithms without linking an unrelated key. |
| `4.1-controller-node-id-text` (**landed**, PR #23685) | `internal/database/bootstrap.go`, `InsertControllerNodeID`, and changestream/upgrade fixtures: uint64 values bound to a TEXT column. | Bind decimal strings and support the full unsigned range. |
| `4.1-lease-migration-tests` (**landed**, PR #23686) | `domain/lease/state/migration_test.go`: ignored seed errors can invalidate test setup. | Assert setup success and genuinely test held/expired leases with appropriate timestamp fixtures. |
| `4.1-crossmodel-secret-revisions` (**in review**, PR #23688) | `domain/crossmodelrelation/state/model/secrets.go`: grouped queries select an arbitrary consuming unit's revision. | Report a secret if any remote consuming unit is behind and return its latest revision once. Join through the indexed offer connection, with matching schema, migration and removal changes. Local-consumer query portability is split into `4.1-local-secret-revisions`; see the recorded scope below. |
| `4.1-dbaccessor-pool-cleanup` | `internal/worker/dbaccessor/tracker.go`: pools opened during setup or reopen attempts are leaked when a later step fails. | Close every pool the tracker opens but does not retain, including failures setting foreign keys, model initialization and reopen ping; preserve pools successfully transferred to the worker. |
| `4.1-nullable-primary-keys` (**in review**, PR #23710) | Model schema tables with a nullable primary-key column: `application_exposed_endpoint_cidr` / `_space` (`NULL application_endpoint_uuid` means the wildcard endpoint; writers in `domain/application/state/exposed.go`, readers through `v_application_exposed_endpoint`), `charm_container_mount`, and `charm_manifest_base`. SQLite accepts NULLs and does not enforce uniqueness over them. Only the exposure tables store NULL. | Exposure on all endpoints moves to application-wide `application_exposed_space` / `_cidr` tables, mirroring bindings; the endpoint tables require the endpoint. `charm_container_mount` requires the container key; `charm_manifest_base` is keyed on its manifest position. Readers, the API and the `""` wildcard are unchanged. The 4.0.12 import splits exposure rows by scope and removes duplicate wildcard rows. `remove-space --force` no longer fails on exposure to both the removed space and `alpha`. A schema test rejects nullable primary key columns. |
| `4.1-agentbinary-conflict` | `RegisterAgentBinary` in `domain/agentbinary/state/{controller,model}/state.go`: queries after a primary-key conflict inside the same transaction. | Avoid the conflict in SQL (`ON CONFLICT DO NOTHING`, then compare), keeping `AlreadyExists` / `AgentBinaryImmutable`. Test new, repeated and different-object registrations. |
| `4.1-lease-pin-conflict` | `domain/lease/state`, `PinLease`: ignoring a unique violation leaves the transaction unusable on PostgreSQL. | Resolve the pin race in SQL with `ON CONFLICT DO NOTHING`; preserve the existing result for already-pinned leases and test competing pins. |

### Schema lane

The only schema correction ready for implementation is the relation-status
integer extension on `4.1-sqlite-integers`; nullable primary keys are in
review (PR #23710). The binary-key, removal-boolean, keymanager, node-ID and
lease-test changes have landed; do not recreate them.

For the integer extension, inspect schema SQL under
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

The landed branches in the map need no further implementation. The active
reviews are #23680, #23683, #23688, #23691, #23693, #23694 and #23710;
update their base or resolve review feedback on their existing branches
rather than recreating them. New independent implementation tasks are
dbaccessor pool cleanup, agent-binary conflict handling and lease pin
conflict handling. The dbrepl latest-row query rewrite is queued separately
as `4.1-dbrepl-latest-row`, following the review request on PR #23691; it
is not covered by that PR's changestream-only rewrite. Secret latest-revision
work follows the cross-model secret review, and lookup-key `NOT NULL` work
follows #23710.

## Cross-model secret review scope and follow-up

The review follow-up for `juju/juju#23688` keeps the remote any-unit-behind
query fix, indexed offer-connection schema/export/import/recovery changes,
consumer cleanup, and their regression tests together.

The independent local-consumer query portability change is split out onto
`4.1-local-secret-revisions`, based on `9ea82c03bd`, for its own PR. It owns
only `domain/secret/state/state.go` methods
`InitialWatchStatementForConsumedSecretsChange` and
`GetConsumedSecretURIsWithChanges`, and
`TestConsumedSecretsChangesMultipleUnitsAndRevisions` in `state_test.go`.
The regression passes with both the original SQLite SQL and the portable
SQL; the full secret-state race suite and focused lint pass. The
schema-dependent remote-consumer fixture stays in `juju/juju#23688`.
The cross-model review follow-up is committed as `29fc497b6e`; the
extracted local-query change is committed as `369fb64c2d`. The separate
PR is drafted, not published.

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
| `4.1-objectstore-conflicts` (**in review**, PR #23694; sentinel follow-up published, new-head CI pending) | `domain/objectstore/state`: use one conflict clause instead of separate clauses for each hash. | Return the documented `objectstoreerrors.ErrMetadataUUIDAlreadyExists` sentinel when a UUID exists with different hashes or size, distinct from `ErrHashAndSizeAlreadyExists`. Test hash and size-only UUID conflicts, duplicate UUID reuse, unchanged metadata/paths/hints, and wrapped sentinel propagation through both service put methods. |
| `4.1-sql-test-literals` (**landed**, PR #23696) | SQL test helpers and production SQL: single-quoted strings instead of SQLite's double-quoted-string fallback. | Bound the file set; preserve fixture and query meaning and coordinate overlapping owners. |
| `4.1-secret-latest-revision` | `domain/secret/state/state.go`: `updateSecret`, `GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and `getSecretsRevisionExpiryChanges` read ungrouped columns beside `MAX(revision)` and rely on SQLite returning them from the latest-revision row. Select the latest revision explicitly instead. This follows the consumed-secret change in PR #23688. | Show the results are unchanged with several revisions, expiry and rotation rows, and both owner kinds. Tests must pass against the old and the new queries. |
| `4.1-sql-standard-syntax` (**landed**, PR #23695) | Two commits. Equality: replace SQLite's `==` with `=` in `domain/keymanager/state` `DeletePublicKeysForUser`, `domain/storageprovisioning/state` `GetVolumeAttachmentIDs` and `domain/crossmodelrelation/state/model` `ValidateApplicationAndEndpointsForOffer`. Join scope: `domain/removal/state/model` `GetDetachInfoForStorageAttachment` joins a one-row aggregate CTE with a comma and then uses `JOIN ... ON` with the table before the comma; spell that join as `CROSS JOIN`. | No behaviour change. Test that removing a user's keys deletes keys no other model uses and keeps shared ones. Test detach info when the requested attachment is no longer alive and no other attachment is, so the fulfilment count is zero. Confirm no other `==` remains in production SQL. |
| `4.1-lookup-key-not-null` | Single-column `id INT PRIMARY KEY` lookup tables lack `NOT NULL`: 48 model tables (for example `agent_stream`, `life`, `os`) and 17 controller tables (for example `cloud_type`, `lease_type`). Nothing writes a NULL ID; the writers are the schema seeds, `AllowCloudType` (a Go `int`), model import (`INSERT ... ON CONFLICT DO NOTHING`) and recovery restore. But the generated 4.1.0 export types use `ID *int64`, so import and recovery would store a NULL-keyed row from a malformed payload. Declare `id INT NOT NULL PRIMARY KEY`, regenerate the export types and narrow the guard in `domain/schema/primary_key_schema_test.go` to exempt only `INTEGER`. Follows `4.1-nullable-primary-keys`, which adds the guard. | Model export types change from `*int64` to `int64`, so `to_v4_1_0` needs a delta per table (or a shared helper) that rejects a nil 4.0.12 ID. Controller export types exist only at 4.1.0 and need no deltas. Test import and recovery with valid and nil IDs. |

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
New-head CI is running; the previous head's green checks are not evidence
for this follow-up.

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
3. Completed: published only those approved updates and checked new-head
   CI, which is pending. The installed `gh pr edit` failed on deprecated
   classic-project GraphQL fields; the approved description was published
   using `gh api --method PATCH repos/juju/juju/pulls/23694` instead.
   Next: confirm new-head CI completes successfully. No review threads were
   changed; further published edits/resolutions need approval.
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
