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
#23756 and #23759. Current heads and CI must be checked before acting;
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

### Validation without repeated full static analysis

For routine code changes, run the affected tests (with `-race` and stress
when concurrency requires them), format touched Go files with `gci`,
check the diff, and run the repository-configured linters on affected
packages rather than invoking every repository-wide analysis:

```sh
golangci-lint run -c .golangci.yml ./path/to/affected/package/...
go run ./scripts/txncheck/main.go "$PWD"
```

The scoped `golangci-lint` command checks tests but not the whole repo;
the separate `txncheck` walks Go source repository-wide and is cheap
(about 0.6 seconds and 64 MB RSS with a warm cache in the lookup-key
checkout). It can catch transaction-test issues the package linter
misses. Neither command replaces schema, dependency, licence or
vulnerability checks. Check that the binaries are on `PATH`; an absent
tool is not a successful lint run.

`make pre-check` skips static analysis unless invoked as
`STATIC_ANALYSIS=1 make pre-check`. The full run includes other checks
and runs `govulncheck` across `github.com/juju/juju/...` when that tool
is installed; it can take substantial time and memory. Do not repeat
it for each small, local follow-up when affected tests, scoped lint,
txncheck and PR CI provide the needed feedback. Run it locally when
dependencies or relevant vulnerable call paths change, when an
advisory appears, before release/integration milestones, and
periodically to catch drift that scoped checks and CI might miss.
Review the GitHub checks; a skipped or failing check is not a pass.

## Open reviews: finish on existing branches

Do not recreate these implementations. Verify outstanding review requests,
CI and mergeability; validate any follow-up against the current head.

| Branch / PR | Finding | Remaining action / acceptance contract |
|---|---|---|
| `4.1-model-config-precedence`, [#23683](https://github.com/juju/juju/pull/23683) | U7 | Review follow-ups are published; reported checks have no failures or pending jobs. Obtain remaining approval. Preserve authoritative agent-table values, import filtering and one effective value per key. |
| `4.1-crossmodel-secret-revisions`, [#23688](https://github.com/juju/juju/pull/23688) | U11 | Connection-scoped uniqueness and cleanup follow-ups are implemented. Resolve cancelled macOS/failed aggregate CI and obtain approval. Keep any-unit-behind semantics, indexed connections, schema/import/recovery/cleanup and regressions together. |
| `4.1-operation-query-portability`, [#23693](https://github.com/juju/juju/pull/23693) | X34/X33 | Commit identity/message and rebase requests are addressed. Resolve cancelled macOS/failed aggregate CI and complete review. Preserve filtering and the reviewed NULL-as-zero column-size fix, which changes the old whole-row-ignored estimate. |
| `4.1-nullable-primary-keys`, [#23710](https://github.com/juju/juju/pull/23710) | U15, charm | Finish review and resolve the cancelled macOS/aggregate CI failure; land charm first. Required container key, manifest-position key and duplicate-position diagnostics/tests are implemented. |
| `4.1-application-exposure-scope`, [#23723](https://github.com/juju/juju/pull/23723) | U15, exposure | Reported checks have no failures or pending jobs; obtain approval. Rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors. |
| `4.1-lookup-key-not-null`, [#23724](https://github.com/juju/juju/pull/23724) | U17 | Reported checks have no failures or pending jobs; obtain approval. Schema-derived decoder inventory, direct `NotValid` classification and the transaction-test fix are implemented. Retain explicit typed deltas and the 65-key scope; later reconcile the combined guard with exposure. |
| `4.1-agentbinary-conflict`, [#23728](https://github.com/juju/juju/pull/23728) | U16/X18 | Shared registration-state follow-up is published at `2618d146da`; reported checks have no failures or pending jobs. Obtain approval. Preserve `AlreadyExists` / `AgentBinaryImmutable`, idempotency and transaction usability. The PoC cherry-picks predate this refactor. |
| `4.1-lease-pin-conflict`, [#23729](https://github.com/juju/juju/pull/23729) | X18 | Implementation, pin-contract documentation and retry-safe test-helper fix are published. Reported checks have no failures or pending jobs; obtain approval. Preserve competing-pin, repeated-pin and missing-lease behaviour. |
| `4.1-sequence-upsert-target`, [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | Common-allocator/RETURNING refactor and tests are committed and published at `76b4a04f09`. Triage unit-test, bundles/revision smoke and aggregate CI failures, then complete review and functional QA. Preserve zero-based allocation, contiguous batches, namespace isolation, rollback and import continuity. |
| `4.1-drop-iif`, [#23751](https://github.com/juju/juju/pull/23751) | X7 | Triage revision-smoke/aggregate CI failures and obtain approval. Preserve the equivalent null-check flags in the connection query and controller views. Complete the documented functional QA and reconcile the final upstream result with the PoC separately. |
| `4.1-cte-membership-subqueries`, [#23750](https://github.com/juju/juju/pull/23750) | X24 | Rebased head `02e9f73e49` has no failing or pending reported checks; obtain approval. Preserve explicit single-column CTE membership and scoped cleanup, lifecycle and departure semantics. |
| `4.1-pr-description-guidance`, [#23755](https://github.com/juju/juju/pull/23755) | Agent guidance | Monitor review and CI and obtain approval. At `15119bb898` the PR is mergeable and review is required; 9 checks passed and 12 were skipped on 2026-11-05. Keep reviewer context, prospective merge-message guidance and reproducible reviewer QA distinct from review history. |
| `4.1-lxd-default-bases`, [#23756](https://github.com/juju/juju/pull/23756) | Integration-test diagnostics and dqlite CI | Head `23163fed56` adds the keymanager fix and real-dqlite domain CI with two temporary exclusions. Unit, default-bases smoke and aggregate checks passed at the recorded snapshot; obtain review. Preserve SSH-key diagnostics and failure status under `set -e`. |
| `4.1-dqlite-nullable-boolean`, [#23759](https://github.com/juju/juju/pull/23759) | Real-dqlite domain CI | Stacked on #23756 at `23163fed56`; follow review and CI at head `f0408a92ba`, then rebase onto `4.1` after #23756 lands. Confirm the full domain step and its CI runtime; preserve NULL/false/true and the upstream-fix sentinel. |

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
`f0408a92ba` as a temporary SQL workaround, stacked on #23756. It casts
the Loki state read and the machine test projections to INTEGER, re-enables
both excluded packages in the real-dqlite CI step, and adds a real-dqlite
sentinel that fails once NULL is preserved by the bundled library.
The workaround does not fix other nullable Boolean reads.

Worktree `juju-4.1-dqlite-nullable-boolean` uses the shared `_deps` and is
clean at commit `f0408a92ba55f6792f319587ac1bd133b60f908c`. Its parent is
#23756 head `23163fed56f7172ff2d4636ad6a8593d4867ea61`; upstream `4.1`
was `08fafe7f49ee39bed52c7288e86be0df6af4adcf` when work began. #23759 is
open, mergeable and requires review; its unit and aggregate CI were still
running at the 2026-11-05 snapshot.

Validation:

- The three reported failures reproduced on real dqlite. Focused
  NULL/false/true cases passed on real dqlite and SQLite for the Loki
  pointer and machine `sql.Null[bool]` projections.
- `make run-go-tests TEST_PACKAGES='./domain/...'
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

- [ ] Complete #23756 review, then rebase #23759 onto `4.1` and finish its
  review. Confirm its full real-dqlite domain CI step succeeds within the
  intended runtime budget. #23756's earlier domain step took about five
  minutes on CI with the two packages excluded.
- [ ] When Juju bundles a release containing #883, remove the temporary
  casts, the workaround-specific projection test and the sentinel.
  Verify NULL, false and true remain distinct in real-dqlite and SQLite
  for nullable Boolean pointers and `sql.Null[bool]`.

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

Not claimed. #23720 has landed; #23688 remains open. Check its current
head and file ownership before dispatching this independent rewrite. A
PoC implementation exists (U19 in UPSTREAM-FIXES.md); an implementer may
read it but must validate on 4.1 independently.

### Further candidates from PoC Stage 5 (2026-11-05)

Not yet scoped as tasks; see UPSTREAM-FIXES.md:

- U20: boolean literals in two network queries and an aggregate in the
  agentpassword machine check. Portability only.
- U21: `unit_state_relation.key` is `TEXT` while Go binds integers.
  Implemented on Stage 5 (`1afd3b9434`) by binding text in Go; no schema
  or export change.
- U22: remaining bare grouped columns in secretbackend, removal and
  storage state. Implemented on Stage 5; secretbackend and removal are
  portability only, storage is a latent arbitrary-row bug.

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
reconciliation of the D43 cherry-picks of #23710, #23723 and #23728
(U15/U16) when they land. Reconciled 2026-11-05 onto `5259edff43`:
#23720, #23726 and #23694's UUID-conflict sentinel; U15/U16 cherry-picked
(see the register's reconciliation record). Keep all other landed-fix reconciliation status in the register;
upstream landing alone does not mark it complete.

## Dispatch

Supply the upstream base, one named branch/worktree, allowed paths,
expected behaviour, prerequisites and required regression cases. The
executor reads this guide and the target repository's instructions; task
selection does not authorise work on other branches.

Example assignment:

> In stock upstream Juju 4.1 at <base>, implement <named pending task>
> from UPSTREAM-PLAN.md on <branch> in <worktree>. Use only that
> checkout's code and test infrastructure. Add the specified regressions,
> run its normal validation, and report the diff and results. Stop for
> unapproved semantics or dependencies.

## Upstream validation and review

- Reproduce the upstream defect where applicable, then add the regression
  alongside the fix. For a behaviour-preserving SQL improvement, assert the
  existing contract across relevant edge cases.
- Run focused tests first, then affected package suites for the assigned
  paths. Preserve unrelated behaviour; do not expand into landed fixes.
- Use the target checkout's standard test commands, dependencies and
  database fixtures. Include actual dqlite checks where required, and its
  prescribed race/stress, watcher-readiness, formatting and lint checks.
- For schema changes, confirm the release's base-DDL/incremental-upgrade
  policy. Use `make rebuild-export` from each branch's own schema; regenerate
  after conflicts from the combined schema, not whole-file selection.
  Exercise writers -> storage -> export -> legacy import -> consumers,
  recovery and supported upgrades. Generation alone is not acceptance;
  generated output must be limited to intended changes.
- Record the base, final commit, exact commands/build tags and results.
  State tests not run and unresolved risks; do not claim broader coverage.
  Explain the upstream defect/benefit and any behaviour change in the PR,
  following the target repository's template.

Return the branch, commits, test evidence and review link to the coordinator.
The coordinator updates shared planning records; parallel agents need not
edit them. Respect the host's test resource budget and other owners' state.
Each independent review targets upstream 4.1; rebase/regenerate when needed
after related changes land, without combining unrelated reviews.
