# Stock upstream Juju 4.1 changes

This is a standalone implementation plan for a stock upstream 4.1
checkout. Read this guide's shared rules, the assigned task, and the target
repository's root/nested AGENTS.md and contribution guidance. No other
planning document or project's checkout, configuration or tests is required.

These issues were discovered during a portability proof of concept;
that explains their origin, not how to implement or validate these changes.
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

`4.1-sqlite-integers` already contained integer FK corrections and the
revision-updater application unit-count fix when this plan was written.
Check its current state; do not duplicate those changes. Add the
relation-status integer correction as a separate commit if still missing,
reusing its legacy-ID parsing helpers and extending its import fixtures.
The user chose this grouping rather than a separate relation-status branch.

## Candidate branch map

Names are proposals except the existing integer branch. Each row defines
an upstream problem to solve, not a patch to copy from another branch.

| Branch | Change and entry points | Required result |
|---|---|---|
| `4.1-sqlite-integers` (existing) | `relation_status_type.id` and `relation_status.relation_status_type_id`: TEXT declarations disagree with integer writers and lookup values. | INT declarations, matching export types and legacy string-ID conversion; preserve existing unit-count fixes. |
| `4.1-binary-key-columns` | `user_activation_key.activation_key`, eight `bakery_config` key columns and `macaroon_root_key.root_key`: raw bytes declared TEXT. | BLOB declarations and consistent storage/export/recovery representations, including legacy compatibility. |
| `4.1-removal-force-boolean` | `removal.force`: INT declaration despite boolean writers/readers. | BOOLEAN declaration, boolean export type and validated 4.0.12 integer conversion. |
| `4.1-model-config-precedence` | `v_model_config` and `domain/modelconfig/state`: duplicate agent stream/version rows let row ordering choose the winner. | Authoritative `agent_version` values win regardless of config contents or row order. |
| `4.1-keymanager-insert-id` | `domain/keymanager/state`, `ensureUserPublicKey`: a zero-row insert can leave `LastInsertId` referring to an earlier key. | Use the inserted row's returned ID; reject unknown algorithms without linking an unrelated key. |
| `4.1-controller-node-id-text` | `internal/database/bootstrap.go`, `InsertControllerNodeID`, and changestream/upgrade fixtures: uint64 values bound to a TEXT column. | Bind decimal strings and support the full unsigned range. |
| `4.1-lease-migration-tests` | `domain/lease/state/migration_test.go`: ignored seed errors can invalidate test setup. | Assert setup success and genuinely test held/expired leases with appropriate timestamp fixtures. |
| `4.1-crossmodel-secret-revisions` | `domain/crossmodelrelation/state/model/secrets.go`: grouped queries select an arbitrary consuming unit's revision. | Defined, tested multi-unit revision semantics; design approval required before implementation. |

### Schema lane

Use one owner to prepare the integer extension, binary-key and removal
boolean branches sequentially; their reviews can proceed concurrently.
The relation-status extension uses parsing helpers on the existing integer
branch. Binary-key and removal-boolean branches can each start from `B`;
neither requires the other's behaviour. The model-config view fix can
proceed independently.

Inspect schema SQL under `domain/schema/{model,controller}/`, generated
export types under `domain/export/types/`, and the legacy conversion code
under `domain/modelimport/transformer/transforms/to_v4_1_0/`. Integer and
boolean work overlaps model export/transform/recovery output; key columns
also affect controller exports, `domain/schema/storage.go` and controller
recovery decoding. Assign shared handwritten changes explicitly.

Regenerate from each branch's own schema using its export/import/recovery
tooling (currently `make rebuild-export`). After a merge conflict,
regenerate from the combined schema rather than selecting one branch's
entire generated file. Confirm the target release's schema-change policy:
edit beta-only declarations where permitted, otherwise use its incremental
patch mechanism and test supported upgrades.

Required checks:

- Integer IDs: valid, nullable where supported, and invalid legacy string
  representations through transform/import/export and persistence.
- Key columns: arbitrary bytes, including NUL/non-UTF-8, through storage,
  export and recovery; preserve supported historical encodings.
- Removal flag: 0/1 to false/true conversion, rejection of other values,
  and persistence/round-trip behaviour.

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
- **Cross-model secrets:** trace both query results to their consumers.
  Decide whether any/all consuming units being behind should trigger an
  update, which revision is returned and how optional revision filtering
  interacts with that rule. Obtain approval before changing semantics.
  Cover no consumers, current/behind consumers, mixed multi-unit revisions,
  multiple secrets/revisions, application-name prefix collisions and
  optional filtering. Assert membership, chosen revisions and duplicates;
  a single-unit test does not establish the contract.

## Suggested first batch

1. **Schema owner:** extend the existing integer branch with relation-status
   IDs; then prepare binary-key and removal-boolean branches one at a time.
2. **Leaf owner A:** keymanager insert-ID correctness.
3. **Leaf owner B:** unsigned controller-node ID binding and fixtures.

Model-config precedence and lease-test reliability can follow as capacity
frees. Cross-model design can proceed concurrently, but implementation
waits for semantic approval. Broad test-literal cleanup follows overlapping
functional work, or uses explicitly reserved files. This keeps reviews
small without creating a dependency chain between unrelated fixes.

## Follow-on SQL improvements

Assign these separately after confirming the upstream benefit and behaviour
contract. Portability is a valid rationale, but not permission to combine
unrelated changes or label observable behaviour changes as mechanical.

| Proposed branch | Change / entry point | Review requirement |
|---|---|---|
| `4.1-changestream-latest-row` | `internal/changestream/stream/stream.go`: replace bare-column `MAX(id)` grouping with explicit latest-ID selection and a join to its row. | Preserve full-row consistency, high-water marks, coalescing and pruning; add regressions and required concurrency coverage. |
| `4.1-blockcommand-order` | `domain/blockcommand/state`: replace implicit `rowid` ordering with explicit block-type order. | Approve the observable ordering change and test it. |
| `4.1-operation-query-portability` | `domain/operation/state`: boolean parameter comparisons and text-based size expressions. | Separate commits; preserve filtering and pruning thresholds, including value-type coverage. |
| `4.1-objectstore-conflicts` | `domain/objectstore/state`: use one conflict clause instead of separate clauses for each hash. | Agree reused-UUID error semantics; test hashes, size mismatches, duplicate UUIDs and unchanged existing metadata. |
| `4.1-sql-test-literals` | SQL test helpers: single-quoted strings instead of SQLite's double-quoted-string fallback. | Bound the file set; preserve fixture meaning and coordinate overlapping owners. |
| `4.1-secret-latest-revision` | `domain/secret/state/state.go`: `updateSecret`, `GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and `getSecretsRevisionExpiryChanges` read ungrouped columns beside `MAX(revision)` and rely on SQLite returning them from the latest-revision row. Select the latest revision explicitly instead. This follows the consumed-secret change in juju/juju#23688. | Show the results are unchanged with several revisions, expiry and rotation rows, and both owner kinds. Tests must pass against the old and the new queries. |
| `4.1-sql-standard-syntax` | Two commits. Equality: replace SQLite's `==` with `=` in `domain/keymanager/state` `DeletePublicKeysForUser`, `domain/storageprovisioning/state` `GetVolumeAttachmentIDs` and `domain/crossmodelrelation/state/model` `ValidateApplicationAndEndpointsForOffer`. Join scope: `domain/removal/state/model` `GetDetachInfoForStorageAttachment` joins a one-row aggregate CTE with a comma and then uses `JOIN ... ON` with the table before the comma; spell that join as `CROSS JOIN`. | No behaviour change. Test that removing a user's keys deletes keys no other model uses and keeps shared ones. Test detach info when the requested attachment is no longer alive and no other attachment is, so the fulfilment count is zero. Confirm no other `==` remains in production SQL. |

Further candidates need named files and tests before dispatch: explicit
upsert targets/qualification, explicit CTE membership subqueries and
`CASE` in model connection queries.
Conflict-handling changes must be split by operation and preserve
idempotency, retries and errors. Do not mix a production time-calculation
or aggregation rewrite into a syntax-only cleanup.

## Dispatch

Supply the upstream base, one named branch/worktree, allowed paths,
expected behaviour, prerequisites and required regression cases. The
executor reads this guide and the target repository's instructions; task
selection does not authorise work on other branches.

Example assignment:

> In stock upstream Juju 4.1 at <base>, implement the keymanager insert-ID
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
  `domain/changestream/state` and `domain/upgrade/state`; `domain/lease/state`;
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
