# Stock upstream Juju 4.1 remaining work

Use this standalone queue for a stock upstream 4.1 checkout. Read the shared
rules, assigned task and target repository's AGENTS.md/contribution guidance;
no PoC checkout, configuration or test setup is required.

Check upstream code, GitHub reviews and CI before acting; implementation,
landing and PoC reintegration are different states. Keep this document
focused on pending actions.
[UPSTREAM-DONE.md](UPSTREAM-DONE.md) records implemented work and rationale;
[UPSTREAM-FIXES.md](UPSTREAM-FIXES.md) owns finding/provenance/status mappings.
Neither is prerequisite reading for an assigned task.

The review register contains a 2026-11-05 snapshot and later updates for
#23756, #23759 and #23765. Current heads and CI must be checked before acting;
implemented scope and historical validation are in UPSTREAM-DONE.md. Skipped
checks are not passes, and publication does not establish QA acceptance.

## Working agreement

- Refresh and record the upstream 4.1 base (`B`). Check current code and
  existing reviews for completed work before implementing anything.
- Start each independent branch from `B`; continue open reviews on their
  existing branches. Do not duplicate landed work or reuse its branch for
  unrelated follow-ups. The charm/exposure and #23756/#23759 stacks are
  documented exceptions.
- Dedicated worktrees are allowed. Assign one branch/worktree per owner;
  never switch, reset or rebase another owner's checkout. A worktree's
  presence does not imply permission to modify it.
- Share existing dependencies with a `_deps` symlink to `../juju/_deps`
  when creating a worktree; do not redownload them unnecessarily.
- Keep one review concern per branch. Multiple commits are fine when they
  serve that concern. Shared generated files are an integration conflict,
  not automatically a reason to stack independent reviews.
- Implement using the APIs, schema layout, helpers and test infrastructure
  present in the target branch. Missing helpers require a small local
  solution or clarification, not dependencies on another project.
- Stop for unresolved behaviour, unexpected dependencies or ownership
  conflicts. Do not weaken tests to make a patch pass.

For dispatch and validation details, read the
[breakout validation guide](.github/skills/upstream-breakout/references/validation.md)
when starting or reviewing implementation work.

## Breakout claims

This table is the authoritative local claim list for new breakouts. Record
the task, owner/session, branch, worktree, base, status and next action.
Keep blocked and paused work claimed; do not assume an existing worktree is
available. Use `reserved` until the worktree and base are confirmed, then
`in progress`, `blocked`, or `published` as appropriate.

| Task | Owner/session | Branch | Worktree | Base | Status | Next action |
|---|---|---|---|---|---|---|
| U22 filesystem attachment row | `fc0b395c-35d9-4277-ba1d-3bc4779db286` | `4.1-storage-filesystem-attachment-row` | `juju-4.1-storage-filesystem-attachment-row` | `148e09f015fe59237f382b9df1a468c15ee7fa42` | published | [#23766](https://github.com/juju/juju/pull/23766), head `65dbcae6a9c3f117948ef667c43d195ad8a38b09`; monitor review and CI, complete live QA. SQLite and dqlite race suites, scoped lint and txncheck passed. |

## Open reviews: finish on existing branches

Do not recreate these implementations. Verify outstanding review requests,
CI and mergeability; validate any follow-up against the current head.

| Branch / PR | Finding | Remaining action / acceptance contract |
|---|---|---|
| `4.1-model-config-precedence`, [#23683](https://github.com/juju/juju/pull/23683) | U7 | Review follow-ups are published; reported checks have no failures or pending jobs. Obtain remaining approval. Preserve authoritative agent-table values, import filtering and one effective value per key. |
| `4.1-crossmodel-secret-revisions`, [#23688](https://github.com/juju/juju/pull/23688) | U11 | Connection-scoped uniqueness and cleanup follow-ups are implemented; obtain approval. The cancelled macOS job caused the aggregate failure; it is not being treated as an investigation task, but may still block merging. Keep any-unit-behind semantics, indexed connections, schema/import/recovery/cleanup and regressions together. |
| `4.1-operation-query-portability`, [#23693](https://github.com/juju/juju/pull/23693) | X34/X33 | Commit identity/message and rebase requests are addressed; complete review. The cancelled macOS job caused the aggregate failure; it is not being treated as an investigation task, but may still block merging. Preserve filtering and the reviewed NULL-as-zero column-size fix, which changes the old whole-row-ignored estimate. |
| `4.1-nullable-primary-keys`, [#23710](https://github.com/juju/juju/pull/23710) | U15, charm | Finish review and land charm first. The cancelled macOS job caused the aggregate failure; it is not being treated as an investigation task, but may still block merging. Required container key, manifest-position key and duplicate-position diagnostics/tests are implemented. |
| `4.1-application-exposure-scope`, [#23723](https://github.com/juju/juju/pull/23723) | U15, exposure | Reported checks have no failures or pending jobs; obtain approval. Rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors. |
| `4.1-lookup-key-not-null`, [#23724](https://github.com/juju/juju/pull/23724) | U17 | Reported checks have no failures or pending jobs; obtain approval. Schema-derived decoder inventory, direct `NotValid` classification and the transaction-test fix are implemented. Retain explicit typed deltas and the 65-key scope; later reconcile the combined guard with exposure. |
| `4.1-lease-pin-conflict`, [#23729](https://github.com/juju/juju/pull/23729) | X18 | Implementation, pin-contract documentation and retry-safe test-helper fix are published. Reported checks have no failures or pending jobs; obtain approval. Preserve competing-pin, repeated-pin and missing-lease behaviour. |
| `4.1-sequence-upsert-target`, [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | Common-allocator/RETURNING refactor and tests are committed and published at `76b4a04f09`. Triage unit-test, bundles/revision smoke and aggregate CI failures, then complete review and functional QA. Preserve zero-based allocation, contiguous batches, namespace isolation, rollback and import continuity. |
| `4.1-drop-iif`, [#23751](https://github.com/juju/juju/pull/23751) | X7 | Triage revision-smoke/aggregate CI failures and obtain approval. Preserve the equivalent null-check flags in the connection query and controller views. Complete the documented functional QA and reconcile the final upstream result with the PoC separately. |
| `4.1-cte-membership-subqueries`, [#23750](https://github.com/juju/juju/pull/23750) | X24 | Rebased head `02e9f73e49` has no failing or pending reported checks; obtain approval. Preserve explicit single-column CTE membership and scoped cleanup, lifecycle and departure semantics. |
| `4.1-pr-description-guidance`, [#23755](https://github.com/juju/juju/pull/23755) | Agent guidance | Monitor review and CI and obtain approval. At `15119bb898` the PR is mergeable and review is required; 9 checks passed and 12 were skipped on 2026-11-05. Keep reviewer context, prospective merge-message guidance and reproducible reviewer QA distinct from review history. |
| `4.1-lxd-default-bases`, [#23756](https://github.com/juju/juju/pull/23756) | Integration-test diagnostics and dqlite CI | Head `63a93201b5` includes the keymanager fix, dqlite RETURNING sentinel and SQLite WAL-mode assertion. Obtain review; preserve SSH-key diagnostics and failure status under `set -e`. |
| `4.1-dqlite-nullable-boolean`, [#23759](https://github.com/juju/juju/pull/23759) | Real-dqlite domain CI | Head `a848d4426e` is based on #23756 at `63a93201b5`; finish review and rebase onto `4.1` after the parent lands. Unit Tests passed with the full dqlite domain step; the retried upgrade pre-check and aggregate check failed. Preserve NULL/false/true and the upstream-fix sentinel. |
| `4.1-controller-export-null`, [#23765](https://github.com/juju/juju/pull/23765) | Controller backup NULL fidelity | Independent PR based directly on `4.1` at `148e09f015`, head `2e12c3150a`. Finish review, monitor CI and complete the PR's disposable-controller backup QA. Preserve nullable Boolean and timestamp values without changing the export format. |

### Nullable primary-key review split

The nullable primary-key stack is charm #23710 -> exposure #23723. Exposure owns
the combined primary-key guard; lookup #23724 owns an independent INT-key
guard. When exposure and lookup meet, regenerate the combined schema,
remove the global guard's INT exemption (retain only single-column INTEGER
rowid keys), and run both guards. Do not tighten it before lookup keys land
or add the other concern to an independent PR. Both U15 fixes are required
for PoC bootstrap reintegration; charm alone is insufficient.

Remote-secret migration contract: preserve valid tracking; warn/discard
only unresolvable revision-tracking metadata, including malformed names.
Never generalize this exception to secret content, grants or core model data.

## Follow-up tasks

Proposed branch names below do not grant permission to modify existing owners'
worktrees. Start new independent work from refreshed `B`, except where an
existing review or specific sequencing is noted. Confirm each site is still
outstanding.

### TODO: dqlite error reporting for `RETURNING`

Follow up independently of the targeted keymanager fix in
[juju/juju#23756](https://github.com/juju/juju/pull/23756). Other Juju queries,
including the shared sequence allocator, need portable `RETURNING` support;
do not treat this defect as a general prohibition on `RETURNING`.

The default-bases investigation reproduced a dqlite 1.18.6 error-response
bug: `query__batch` writes column metadata before stepping the statement.
When an `INSERT ... RETURNING` encounters a stale-snapshot write conflict,
the failure path appends an error without clearing the metadata. The client
reads the column count as the error code and Sqlair's `_sqlair_0` alias as
the message, hiding the retryable database error.

- [x] Locate the upstream fix:
  [canonical/dqlite#918](https://github.com/canonical/dqlite/pull/918) merged
  into `main` on 2026-11-04 as `366191c0d3f0`. It discards partial query
  responses on error and tests prepared/direct `RETURNING` constraint
  failures. Juju's bundled dqlite 1.18.6 does not contain it.
- [ ] Identify a release or approved backport containing that fix and
  integrate it into Juju's dqlite dependency.
- [ ] Verify Juju's actual dqlite dependency preserves retryable errors from
  `RETURNING` under contention, as well as successful returned IDs and
  zero-row results. Keep SQLite and PostgreSQL portability intact.
- [ ] Review other `RETURNING` call sites for exposure to this defect.
  Reconsider the keymanager insert/select workaround after the dependency
  fix is available and validated.

Related but distinct: open
[canonical/dqlite#920](https://github.com/canonical/dqlite/pull/920) preserves
busy errors otherwise misreported as a nonempty statement tail. Do not
assume the response-buffer fix also resolves that path.

Reproducer: `domain/keymanager/state/state_test.go`,
`TestEnsureUserPublicKeyConcurrentWrite`, using real dqlite. With the original
`RETURNING` implementation it produces the same `_sqlair_0` error as
[the failing CI job](https://github.com/juju/juju/actions/runs/42120009349/job/127569657575).
For now, keep the current fix limited to keymanager's insert followed by a
unique-key lookup in the same transaction; do not restore `LastInsertId`.

### Nullable BOOLEAN results in real-dqlite domain tests

The broader real-dqlite `-race` run for
[juju/juju#23756](https://github.com/juju/juju/pull/23756) exposed three
existing failures where SQL NULL Boolean values are returned as non-null:

- `domain/logging/state`: `TestSetLokiConfigInsecureSkipVerifyNil` expects
  `InsecureSkipVerify == nil`, but receives a non-nil pointer.
- `domain/machine/state`:
  `TestDetachLostMachineCloudInstanceCleansVolumeAndPreservesIntent` and
  `TestDetachLostMachineCloudInstanceCleansFilesystemAndPreservesIntent`
  expect cleared `persistent` / `read_only` columns to scan into invalid
  `sql.Null[bool]` values, but receive valid values.

The root cause is dqlite's response encoding, fixed upstream by
[canonical/dqlite#883](https://github.com/canonical/dqlite/pull/883),
but not in the bundled 1.18.6. This is separate from the `RETURNING` defect.

Published [juju/juju#23759](https://github.com/juju/juju/pull/23759) at
`a848d4426e` as a temporary SQL workaround, based on #23756 at `63a93201b5`.
It casts the logging and provisioner Loki reads, both storage-status
projections, and machine test projections to INTEGER. It re-enables both
excluded packages in the real-dqlite CI step and adds a sentinel that fails
once NULL is preserved by the bundled library. The status output remains
false for both NULL and false; the provisioner's nil Loki pointer retains
the distinction between the default and explicit false.

The independent [juju/juju#23765](https://github.com/juju/juju/pull/23765),
head `2e12c3150a` directly on upstream `4.1` at `148e09f015`, preserves
NULL Boolean and datetime/timestamp pointers in controller exports. It
adds `IS NULL` markers to existing queries, following model export, without
changing the backup format. Its transaction retry resets the captured
payload. The worktrees are `juju-4.1-dqlite-nullable-boolean` and
`juju-4.1-controller-export-null`; both were clean at these heads.

Validation:

- The three reported failures reproduced on real dqlite. Focused
  NULL/false/true cases passed on real dqlite and SQLite for the Loki
  pointer and machine `sql.Null[bool]` projections.
- At the earlier #23759 snapshot, `make run-go-tests TEST_PACKAGES='./domain/...'
  TEST_ARGS='-p 2 -race -count=1'` passed for 225 test packages with
  `dqlite,libsqlite3` tags in 1067 s. After a final table-alias-only SQL
  edit, the complete logging package passed with `-race` on both backends.
- The sentinel passed with the uncast query and failed with its intended
  removal message when temporarily changed to return a preserved NULL;
  the uncast query was restored.
- Dqlite REPL QA confirmed the stored NULL reads as false uncast, while
  NULL/false/true cast to NULL/0/1.
- Scoped `golangci-lint`, repository `txncheck`, `gci` and
  `git diff --check` passed. `make pre-check` passed with static analysis
  skipped; scoped lint was run separately. PostgreSQL was not tested.
- At the current heads, affected #23759 packages and #23765's controller
  export suite passed on SQLite and real dqlite; dqlite tests used `-race`.
  Export generator tests, regeneration without drift, scoped lint and
  repository `txncheck` passed. The manual backup QA in #23765 is pending.
- On 2026-11-06, #23759's Unit Tests job passed on its second attempt,
  including logging and machine in the full real-dqlite domain step
  (approximately 5m25s). Its upgrade pre-check failed again on Ubuntu 26.04
  at `snap info juju` with `no snap found for "juju"`; the aggregate check
  failed. A local `snap info juju` succeeded, but the runner failure's
  underlying cause is not confirmed. #23765 CI was still running without
  reported failures; its upgrade smoke was skipped.

- [ ] Complete #23756 review, then rebase #23759 onto `4.1` and finish its
  review. Resolve or retry the failed upgrade pre-check and aggregate CI;
  continue monitoring the real-dqlite domain step's runtime.
- [ ] Finish #23765 review, CI and functional backup QA independently of
  #23759. Retain the distinct NULL, false, true and timestamp cases.
- [ ] When Juju bundles a release containing #883, review the temporary
  casts, sentinel and model/controller export NULL markers; sweep nullable
  Boolean reads. Verify NULL, false and true remain distinct where required,
  along with nullable timestamps, on real dqlite and SQLite.

### `4.1-install-dependencies` — developer tooling

On a separate branch, update `make install-dependencies` to install or verify
the tools needed for the repository's normal build, formatting, tests and
static analysis: `shellcheck`, `gci`, `golangci-lint`, `stress`, and
`libsqlite3-dev`. The current target already installs `libsqlite3-dev`, but
running `STATIC_ANALYSIS=1 make pre-check` without it fails compiling
`go-sqlite3` (`sqlite3.h` missing); without `shellcheck`, the test harness
stops before linting. Static analysis also reports `shfmt` missing (shell
analysis disabled), `go-licenses` missing (licence analysis skipped), and
`govulncheck` missing (vulnerability analysis disabled).
Inventory other required or skipped tools and include them where appropriate,
using the versions and installation conventions supported by the target
branch. Ensure installed executables are on the PATH used by checks.
Do not silently require new privileges or mix this tooling change into the
operation query review. Validate on a clean environment and a repeated
invocation, then run the checks that previously failed or were skipped.

### `4.1-secret-latest-revision`

After the remaining consumed-secret review (#23688), independently rewrite
five queries in `domain/secret/state/state.go`: `updateSecret`,
`GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and
`getSecretsRevisionExpiryChanges`, and the same pattern in
`domain/unitstate/state/commithook.go` `updateCharmSecret`. Explicitly
select the latest revision instead of relying on ungrouped columns beside
MAX, and order revision listings by revision number. Existing and new SQL
must pass regressions with multiple revisions, expiry/rotation rows and
both owner kinds. This does not duplicate the two local queries landed in
#23720.

#23720 has landed; #23688 remains open. Check its current
head and file ownership before dispatching this independent rewrite. A
PoC implementation exists (U19 in UPSTREAM-FIXES.md); an implementer may
read it but must validate on 4.1 independently.

### Further break-outs from PoC Stage 5 (2026-11-05)

Each is implemented and tested on `pg-poc/05-queries` (published
2026-11-06). An implementer may read the PoC commit but must reimplement
or cherry-pick onto 4.1 and validate there independently. None depends on
the PostgreSQL harness; the regressions run on SQLite and dqlite. Keep one
branch per bullet unless review asks to combine them. Check the breakout
claims above before dispatch.

#### `4.1-storage-filesystem-attachment-row` — U22, latent bug (priority)

Published as [#23766](https://github.com/juju/juju/pull/23766); continue
review and CI on the existing branch. Live QA remains pending.

`domain/storage/state/instance.go` `GetStorageInstanceInfo` joins every
filesystem attachment on the unit's net node and collapses the fan-out
with `GROUP BY sa.uuid`. The schema does not forbid two filesystem
attachments for the same filesystem and net node, so dqlite returns an
arbitrary mount point. Pick one row deterministically (PoC: the lowest
attachment UUID via a correlated subquery), drop the `GROUP BY`, and move
the `ORDER BY` out of the `FROM` subquery in both the attachment and the
block-device link queries. Regression: two filesystem attachments on the
same node; the PoC test failed 7 of 10 runs against the old query.
Question for review: whether the schema should instead forbid the
duplicate (a unique index on `(storage_filesystem_uuid, net_node_uuid)`),
which needs import/export review. PoC: `640d3bcabf`.

#### `4.1-unitstate-relation-key` — U21, type mismatch

`setUnitStateRelation` binds Go `int` relation IDs to the `TEXT`
`unit_state_relation.key`, and `GetUnitState` scans it back into `int`.
Bind decimal text and parse on read, returning an error for a
non-decimal key; no schema or export change. Existing `TestSetUnitState*`
and `TestUpdateUnitStateRelation` cover it; add a read test with a
non-decimal key. PoC: `1afd3b9434`.

#### `4.1-x16-grouped-columns` — U22, portability

Not a behaviour change on dqlite; the ungrouped columns are functionally
dependent on the grouping keys, or the `GROUP BY` only removes duplicate
join rows.

- `domain/secretbackend/state/state.go`: `ListSecretBackends` and the
  in-use Kubernetes backend query group by every selected column;
  `getK8sSecretBackendForModel` uses `SELECT DISTINCT`. PoC `52a0eea6ee`.
- `domain/removal/state/controller/secret.go`
  `getK8sSecretBackendForModel`: `SELECT DISTINCT` instead of
  `GROUP BY vm.name, vcca.attribute_key`; adds
  `TestGetActiveModelSecretBackendK8sBuiltIn`, which had no coverage
  before. PoC `367f754dd6`.

Follow-up to raise in review, not changed: both
`getK8sSecretBackendForModel` copies return one CA certificate per cloud
even when a cloud has several (the converters keep the last row).

#### `4.1-boolean-and-aggregate-literals` — U20, portability

`domain/network/state` `GetMachineAppBindings` and `GetUnitAPIAddresses`
compare booleans with `0`/`1`; use `FALSE`/`TRUE`. `domain/agentpassword`
`MatchesMachinePasswordHashWithNonce` selects `mci.instance_id` beside
`COUNT` without grouping; use `MAX(mci.instance_id)` (a machine has at
most one cloud instance row). PoC `30e2bca25a`, `ada2a852ee`; the
agentpassword test SQL fix is Stage 6 `fa75871904`.

## Existing implementations and published tracking references

Current published implementation heads are recorded below. Continue these
branches rather than redoing the implementations.

| Branch / tracking reference | Published committed snapshot | Remaining work |
|---|---|---|
| [4.1-drop-iif](https://github.com/jameinel/juju/tree/4.1-drop-iif), [juju/juju#23751](https://github.com/juju/juju/pull/23751) | [adf9f317a6](https://github.com/jameinel/juju/commit/adf9f317a6962f5722bb1f4e401db3e9723fd768): three controller schema-view replacements, following query commit `08b9cc0d7c` | Review, CI triage and functional QA remain. |
| [4.1-cte-membership-subqueries](https://github.com/jameinel/juju/tree/4.1-cte-membership-subqueries), [#23750](https://github.com/juju/juju/pull/23750) | [02e9f73e49](https://github.com/jameinel/juju/commit/02e9f73e4936d16f8acb23aaeae5e35228b315b7): rebased eleven membership expressions and departure-scope regression | Review remains. |
| [4.1-sequence-upsert-target](https://github.com/jameinel/juju/tree/4.1-sequence-upsert-target), [juju/juju#23731](https://github.com/juju/juju/pull/23731) | [76b4a04f09](https://github.com/jameinel/juju/commit/76b4a04f09772f4ccc4d02ec638fe31d13831dc7): shared scalar allocator with RETURNING and tests, following `74aabf93b4` | Review, CI triage and functional QA remain. |

Use these published heads. Historical validation below is not a new check
of the latest heads.

### `4.1-drop-iif` — X7

**Published for review (2026-11-04):** [juju/juju#23751](https://github.com/juju/juju/pull/23751)
targets `4.1` from `4.1-drop-iif`, with base
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` and head
`adf9f317a6962f5722bb1f4e401db3e9723fd768`. The query commit
`08b9cc0d7c` and schema-view commit `adf9f317a6` replace all tracked
`IIF` calls with equivalent `IS NOT NULL` predicates. The schema change
affects `v_model_all.is_controller_model` (also projected by `v_model`),
`v_model_state.migrating` and `v_cloud.is_controller_cloud`. The existing
joins and filters are unchanged. Base-DDL editing was chosen because the
local 4.1 tags contain only `v4.1-beta1` and no controller post-patches
are registered. PoC D32 used CASE; reconciliation must choose.

Existing tests cover the connection-query activation/claim combinations
and not-found path, both controller-model and controller-cloud flag values,
and migrating flags for all import phases. No tests were added.
On the schema-view changes, `make run-go-tests` passed with `-race` for
schema and the cloud, model/controller, status/controller,
removal/controller, secretbackend, keyupdater and controllerupgrader state
packages. Schema, cloud, model/controller and status/controller also passed
uncached (`-count=1`). Controller-model view stress completed 48 runs with
zero failures before the bounded timeout. Scoped `golangci-lint`,
repository `txncheck` and `git diff --check` passed; `make pre-check` used
its default, which skips static analysis. PostgreSQL execution and the PR's
functional migration QA have not been performed.

Remaining: monitor CI and review, complete functional QA, and reconcile
the PoC after landing. The published PR description is also saved locally at
[PR-4.1-drop-iif.md](PR-4.1-drop-iif.md).

## Lookup-key NOT NULL task (separate session)

Continue existing independent #23724, not a new implementation branch.
Scope: single-column `id INT PRIMARY KEY NOT NULL` (48 model, 17 controller
at `8369e6e2a9`), generated scalar IDs and consumed payload boundaries.
Recheck inventory at the actual base; leave INTEGER rowid keys, composite
keys, historical DDL, 4.0.12 export types and unrelated nullable fields alone.
Do not replace explicit schema constraints with generator inference.

Implementation, regression coverage and recorded live migration QA are in
UPSTREAM-DONE.md. Remaining work is review/CI and later combined-guard
reconciliation. Preserve this acceptance contract on further changes:

- Keep the explicit typed-delta methods: a generic loop still needs an ID
  accessor and field-mapping closure in every method, obscuring each
  conversion. `requiredLookupID` already centralises NULL rejection.
  Exercise every consumed legacy lookup type's full row, multiple rows,
  nullable non-ID fields, nil IDs and valid zero. Reject missing IDs with
  table/row context and invalid-payload classification; never invent zero.
  No legacy controller delta is needed.
- In `domain/export/payload.go`, validate consumed 4.1.0 model/controller
  raw YAML IDs before scalar decoding loses NULL/omitted versus zero.
  Test valid zero/non-zero, NULL and missing IDs at the actual boundary;
  do not broaden validation to unrelated scalar fields.
- Preserve import exclusions for `change_log_edit_type` and
  `change_log_namespace`, valid seed IDs and seeded-import conflict handling.
  Test representative legacy decode -> transform -> import -> export
  persistence, including existing seeds and a valid non-seed where allowed.
- Verify fresh model/controller schemas reject NULL keys and the complete
  affected declaration/generated-field set is correct. Exercise generated
  recovery decode/replay, valid dump round trips, NULL/missing rejection
  and failure atomicity without relaxing immutable-table requirements.
- Run affected schema/export/import/recovery/cloud-state race suites,
  focused stress/lint/build and regeneration-idempotence checks. Keep the
  INT-key guard independent; reconcile the global guard as described above.
  Stop for release-policy changes, new nullable-key writers or incompatible
  representations. No PostgreSQL-only work belongs in this review.

## Existing CTE membership implementation — X24

**PR published 2026-11-04:** continue `4.1-cte-membership-subqueries` in
juju/juju#23750; do not dispatch a duplicate X24 implementation.
The original implementation started from upstream/4.1 base
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` and replaced eleven
production expressions in eight files (plus focused regressions):
- `domain/application/state/{application_endpoint,k8sservice}.go`
- `domain/removal/state/model/{application,unit,storage}.go`
- `domain/operation/state/{task,prune}.go`
- `domain/relation/state/relation.go`

#23693 touches `prune.go` for size expressions, not CTE membership; keep
this independent and preserve that review's separate ownership (possible
same-file integration conflict).
Contracts: preserve refresh endpoint filtering, orphan/shared hostname and
pod-device deletion isolation, constraint deletion scope, storage lifecycle
guards, operation completion/size filtering and departed-unit membership.
No schema/generated files, service/API behavior or transaction changes.
Local implementation commit: `8ce0410f515a9b46bc09ba006c108852267eb8e7`.
All eleven expressions use explicit single-column membership subqueries;
existing CTEs, joins, lifecycle guards and transaction behavior are preserved.
Added `TestGetConsumerRelationUnitsChangeDepartureScope` for empty/partial/full
membership and cross-relation/application isolation.
Rebased without conflicts onto refreshed `upstream/4.1`
`5259edff4329ca4a1de80d77e0d614b832d4ad72`; new local head
`02e9f73e4936d16f8acb23aaeae5e35228b315b7`. Range-diff confirms the patch is
unchanged. Published this head to the fork with an exact old-head lease;
verified the remote head. Worktree clean.

Validation with shared `_deps` and stock SQLite/Dqlite fixtures:
- Focused regressions across all four affected state packages passed with
  original and portable SQL (`make run-go-tests`, `-p 2 -race`).
- The new departure-scope regression also passed separately on original SQL.
- All four full state-package race suites passed: application, removal/model,
  operation and relation (`make run-go-tests`, `TEST_ARGS='-p 2 -race'`).
- Final relation package compiled with `-race -c`; bounded departure-scope
  stress completed 38 iterations with zero reported failures (two active
  when outer `timeout 31` stopped the run, exit 124). Binary removed.
- `make pre-check` passed (default skips static analysis); Go lint passed via
  `RUN_SUBTEST=run_go make static-analysis
  STATIC_ANALYSIS_JOB=test_static_analysis_go`.
- `gofmt`, prescribed `gci` and `git diff --check` passed. Inventory found no
  remaining uppercase SQL `IN table/cte` shorthand outside comments.
- After rebasing, all four affected state-package race suites passed again;
  `make juju` built a session-local client, and `juju version` ran successfully.
  The client binary was removed.

Approved PR [juju/juju#23750](https://github.com/juju/juju/pull/23750) is
open, targets `4.1` and has verified head and description. On 2026-11-05 it is
mergeable with review required and no failing or pending reported checks.
Not landed; the PoC retains its earlier equivalent SQL, not a verified
reconciliation of the final upstream result. No local PostgreSQL tests, full
repository suite or live deployment/migration QA were run.

## Sequence review follow-up — X3/X4

Continue existing [juju/juju#23731](https://github.com/juju/juju/pull/23731)
at `76b4a04f09772f4ccc4d02ec638fe31d13831dc7`. The shared scalar allocator,
`ON CONFLICT ... RETURNING` and regressions are committed and published;
there is no outstanding uncommitted implementation.

Preserve both allocator entry points, zero-count no-op, namespace isolation,
zero-based contiguous allocation, rollback and suppressed-write detection.
Concurrent tests must collect results only after the whole transaction
succeeds. The fixtures exercise SQLite/Dqlite contracts, not PostgreSQL SQL
acceptance or execution.

Triage the current unit-test, bundles/revision smoke and aggregate CI
failures, complete the PR's functional QA and obtain review approval.
Historical decisions and validation are in
[UPSTREAM-DONE.md](UPSTREAM-DONE.md#sequence-allocation-and-review-follow-ups).
Other X3/X4 operations remain available for independent scoping.

## Candidates needing bounded scope before dispatch

These register findings are not ready-made assignments. Identify remaining
production paths, behaviour contracts and focused tests before branching;
split conflicts by operation and preserve retries, idempotency and errors.

| Finding | Candidate scope / next action |
|---|---|
| X7 | Published as #23751; follow the existing review above. |
| X24 | Published as #23750; follow the existing review and validation record above. Do not duplicate. |
| X3/X4, other upserts | Sequence allocation is implemented in #23731. Inventory any other remaining operations before dispatch; do not recreate that fix or combine unrelated conflict paths. |
| U11 watcher follow-up | Per-offer-connection remote watcher scoping is separate from #23688's offered-application scope. Agree the event/isolation contract and tests before dispatch. |
| X33 stored-bytes follow-up | A stored-bytes metric such as `pg_column_size` needs backend-aware state code; #23693's text-size work does not implement it. Scope separately, not as a mechanical rewrite. |
| X16 dbrepl output follow-up | Published as [#23757](https://github.com/juju/juju/pull/23757); review, CI and live REPL QA remain. |

### X16 dbrepl output follow-up

Session `1fedb56c-3ba7-4099-8914-c6726b9cd337` used branch
`4.1-dbrepl-output` and worktree `juju-4.1-dbrepl-output`, independently
from upstream 4.1 base `08fafe7f49ee39bed52c7288e86be0df6af4adcf`
(verified 2026-11-05). Scope: `internal/worker/dbrepl/worker.go` and
its existing tests. Extract the shared change-log/change-stream row
scanning and tabwriter output, preserving each query, arguments, output,
resource handling and error reporting. Exercise both commands, including
empty results and failures as appropriate. #23730 is merged; no open
4.1 PR touching this package was found. Other independent tasks remain
available.

Commit `0583a1972cc6ecff30c36d8653c11066162c2770` extracts the shared
renderer without changing either SQL query. Tests cover change-log rows
including repeated keys and zero ID, empty and identical output for both
commands, query-error logging, and the existing change-stream cases.
Published as [#23757](https://github.com/juju/juju/pull/23757); review,
CI and live REPL QA remain outstanding.

Validation on this commit:
- `make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl
  TEST_ARGS='-p 2 -race -count=1'` passed with `dqlite,libsqlite3` tags.
- `go test -race -p 2 -count=1 ./internal/worker/dbrepl` passed with SQLite.
- Scoped `golangci-lint`, repository `txncheck`, `gci`, `gofmt` and
  `git diff --check` passed. `make pre-check` passed with static analysis
  skipped; the scoped linter was run separately.
- Actual-Dqlite race test binary stress for change commands and query-error
  handling completed 20 runs without failures; two were active when the
  bounded `timeout 31` ended it with status 124.
- No live REPL, PostgreSQL, repository-wide test suite or full static
  analysis was run.

Not stock-upstream tasks: the PostgreSQL `constraint.allocate_public_ip`
export-type mismatch needs backend-aware generation; the intentional
`TestSetAPIAddressesSharedPeerRollsBack` NOT NULL failure is not a dqlite
bug. Keep both recorded rather than reopening them as upstream fixes.

Separate documentation review: follow `4.1-pr-description-guidance` in
[#23755](https://github.com/juju/juju/pull/23755), independently of the SQL
changes. The older [patch](4.1-pr-description-guidance.patch) and
[PR description](PR-4.1-pr-description-guidance.md) are historical drafts,
not the published branch or PR description.

## PoC reconciliation handoff

Do not implement PoC integration in stock-upstream worktrees. After landing,
hand merged revisions to the integration owner using UPSTREAM-INTEGRATION.md
and update the register. Explicit outstanding obligations: U17 guard
integration, independent reconciliation of #23688 after it lands, and
reconciliation of the D43 cherry-picks of #23710 and #23723 (U15) when
they land. For `4.1-agentbinary-conflict`, [#23728](https://github.com/juju/juju/pull/23728)
has merged; reconcile its final shared registration-state implementation
with the earlier U16 PoC cherry-picks. Preserve `AlreadyExists` /
`AgentBinaryImmutable`, idempotency and transaction usability. Historical
implementation and merge details are in [UPSTREAM-DONE.md](UPSTREAM-DONE.md).
Reconciled 2026-11-05 onto `5259edff43`:
#23720, #23726 and #23694's UUID-conflict sentinel; U15/U16 cherry-picked
(see the register's reconciliation record). Keep all other landed-fix reconciliation status in the register;
upstream landing alone does not mark it complete.

## Dispatch

Use the [upstream-breakout skill](.github/skills/upstream-breakout/SKILL.md)
for task selection, coordinated claims, implementation and review preparation.
The [validation guide](.github/skills/upstream-breakout/references/validation.md)
also applies to executors not using the skill.
