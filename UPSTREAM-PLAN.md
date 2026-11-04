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

Status checked against GitHub on 2026-11-04: 9 open reviews and 14 merged
PRs. Current heads, merge revisions and the CI snapshot are in
[the register](UPSTREAM-FIXES.md#upstream-review-status); implemented scope
and historical validation are in UPSTREAM-DONE.md.

## Working agreement

- Refresh and record the upstream 4.1 base (`B`). Check current code and
  existing reviews for completed work before implementing anything.
- Start each independent branch from `B`; continue open reviews on their
  existing branches. Do not duplicate landed work or reuse its branch for
  unrelated follow-ups. The charm/exposure stack below is the only exception.
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
| `4.1-model-config-precedence`, [#23683](https://github.com/juju/juju/pull/23683) | U7 | Review follow-ups are published and CI is green; obtain remaining approval. Preserve authoritative agent-table values, import filtering and one effective value per key. |
| `4.1-crossmodel-secret-revisions`, [#23688](https://github.com/juju/juju/pull/23688) | U11 | Connection-scoped uniqueness and cleanup follow-ups are implemented; the refreshed head is mergeable and CI is running. Assess that head and obtain approval. Keep any-unit-behind semantics, indexed connections, schema/import/recovery/cleanup and regressions together. |
| `4.1-operation-query-portability`, [#23693](https://github.com/juju/juju/pull/23693) | X34/X33 | Commit identity/message and rebase requests are addressed; submission checks pass. Triage default-bases smoke and cancelled macOS CI, then complete review. Preserve filtering and the reviewed NULL-as-zero column-size fix, which changes the old whole-row-ignored estimate. |
| `4.1-nullable-primary-keys`, [#23710](https://github.com/juju/juju/pull/23710) | U15, charm | Finish review and resolve the cancelled macOS/aggregate CI failure; land charm first. Required container key, manifest-position key and duplicate-position diagnostics/tests are implemented. |
| `4.1-application-exposure-scope`, [#23723](https://github.com/juju/juju/pull/23723) | U15, exposure | Finish review and triage bundles smoke. Rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors. |
| `4.1-lookup-key-not-null`, [#23724](https://github.com/juju/juju/pull/23724) | U17 | CI is green; obtain approval. Schema-derived decoder inventory, direct `NotValid` classification and the transaction-test fix are implemented. Retain explicit typed deltas and the 65-key scope; later reconcile the combined guard with exposure. |
| `4.1-agentbinary-conflict`, [#23728](https://github.com/juju/juju/pull/23728) | U16/X18 | Implementation and regressions exist in both state packages. Triage default-bases smoke and obtain approval. Preserve `AlreadyExists` / `AgentBinaryImmutable`, idempotency and transaction usability. Do not recreate the fix. |
| `4.1-lease-pin-conflict`, [#23729](https://github.com/juju/juju/pull/23729) | X18 | Implementation, pin-contract documentation and retry-safe test-helper fix are published. Triage bundles/revision smoke and obtain approval. Preserve competing-pin, repeated-pin and missing-lease behaviour. |
| `4.1-sequence-upsert-target`, [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | Assess the published test/comment follow-up at `74aabf93b4`; new-head CI/review has not been assessed. The common-allocator/RETURNING refactor remains uncommitted, not part of this head. Preserve zero-based allocation, contiguous batches, namespace isolation, rollback and import continuity. |

### Nullable primary-key review split

The only approved stack is charm #23710 -> exposure #23723. Exposure owns
the combined primary-key guard; lookup #23724 owns an independent INT-key
guard. When exposure and lookup meet, regenerate the combined schema,
remove the global guard's INT exemption (retain only single-column INTEGER
rowid keys), and run both guards. Do not tighten it before lookup keys land
or add the other concern to an independent PR. Both U15 fixes are required
for PoC bootstrap reintegration; charm alone is insufficient.

Remote-secret migration contract: preserve valid tracking; warn/discard
only unresolvable revision-tracking metadata, including malformed names.
Never generalize this exception to secret content, grants or core model data.

## New implementation tasks

Names below are proposed branches, not permission to modify existing owners'
worktrees. Start independently from refreshed `B`, except the secret task's
review sequencing noted below. Confirm each site is still outstanding.

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
`getSecretsRevisionExpiryChanges`. Explicitly select the latest revision
instead of relying on ungrouped columns beside MAX. Existing and new SQL
must pass regressions with multiple revisions, expiry/rotation rows and
both owner kinds. This does not duplicate the two local queries landed in
#23720.

Not claimed. #23720 has landed; #23688 remains open. Check its current
head and file ownership before dispatching this independent rewrite.

## Existing implementations and published tracking references

Committed local work was published to `jameinel/juju` on 2026-11-04.
Continue these branches rather than redoing the implementations.

| Branch / tracking reference | Published committed snapshot | Remaining local work |
|---|---|---|
| [4.1-drop-iif](https://github.com/jameinel/juju/tree/4.1-drop-iif) | [08b9cc0d7c](https://github.com/jameinel/juju/commit/08b9cc0d7cfa3a0ce260cbd41fbb8118f133094d): model connection-info query | Three schema-view replacements remain uncommitted in the transfer patch. No upstream PR yet. |
| [4.1-cte-membership-subqueries](https://github.com/jameinel/juju/tree/4.1-cte-membership-subqueries) | [8ce0410f51](https://github.com/jameinel/juju/commit/8ce0410f515a9b46bc09ba006c108852267eb8e7): eleven membership expressions and departure-scope regression | No uncommitted code. Review the draft and obtain approval before opening an upstream PR. |
| [4.1-sequence-upsert-target](https://github.com/jameinel/juju/tree/4.1-sequence-upsert-target), [juju/juju#23731](https://github.com/juju/juju/pull/23731) | [74aabf93b4](https://github.com/jameinel/juju/commit/74aabf93b4ec4337caf5344fb553e9e33c3b5f38): successful-transaction result collection and comments | Common-allocator/RETURNING refactor and tests remain uncommitted in the transfer patch; review response remains unpublished. |

The clean CTE worktree was removed; its branch/commits are retained.
The dirty IIF and sequence worktrees remain. Restore instructions and
the exact bundle/patch inventory are in
[the transfer record](UPSTREAM-FIXES.md#transfer-artifacts-and-cleanup).
Historical validation below is not validation of the uncommitted changes.

### `4.1-drop-iif` — X7

**Claimed / in progress (2026-10-30):** Copilot session
`bb796609-3e14-42e7-b27a-cd63eee42f63` owns branch `4.1-drop-iif` and
worktree `/home/jameinel/dev/go/src/github.com/juju/4.1-drop-iif`
(renamed from `4.1-model-connection-info-case`; that name/path is gone).
Base `B` = `e22f5666d0861d3c84e8465c1ff86fbd23f58f48` (upstream/4.1).
Scope widened (approved): every `IIF` in non-dependency code, i.e.
`GetModelConnectionInfo` (`domain/model/state/controller/state.go`) plus
the three controller views in `domain/schema/controller/sql/0007-model.sql`
(`v_model.is_controller_model`, `v_model_state.migrating`,
`v_cloud.is_controller_cloud`). Each becomes the bare `IS [NOT] NULL`
boolean (approved over CASE; PoC D32 used CASE, so reconciliation must
choose). Base-DDL edit: 4.1 has only `v4.1-beta1`, no registered
post-patches and precedent post-beta base edits. No overlap with the X24
claim's files. Plan: one commit per area; run schema suites and view
consumers (cloud, removal, status, controllerupgrader, model, secretbackend)
race tests, stress and lint; update draft PR locally.

Status: query commit `08b9cc0d7c` (one-line `state.go` change, amended from
the CASE version `6b5aa7348d`) is published on the branch linked above,
without an upstream PR. Schema-view commit in
progress: the three view replacements are uncommitted in
`domain/schema/controller/sql/0007-model.sql`, verified on 2026-11-04.
`model_migration_import.model_uuid` is unique, so the join returns at most
one row. Existing tests already cover all four
activated × claim combinations (`TestCheckModelExistsNotActivated`,
`TestGetModelConnectionInfo*`) plus not-found, so no tests were added. They
pass on base SQL. Validation of `08b9cc0d7c`: `make run-go-tests`
focused `-race -count=3` (4 tests × 3 passed; `TEST_FILTER` is unquoted,
so avoid `(` and `|`); `./domain/model/state/... ./domain/model/service/...`
`-race` passed; stress 32 runs/0 failures in 30s. `gofmt`/`git diff --check`
and `test_static_analysis_go` passed. Draft:
[PR-4.1-drop-iif.md](PR-4.1-drop-iif.md).
Remaining: finish and validate the schema-view changes and revise the draft,
then obtain approval to publish those changes and open against `4.1`:
its current text describes only the query and explicitly excludes DDL.

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

**Implemented 2026-10-30 / branch published 2026-11-04:** Copilot session
`767870f3-0997-48f4-a415-c1efab02d088` reserves branch
`4.1-cte-membership-subqueries` and dedicated worktree
`/home/jameinel/dev/go/src/github.com/juju/4.1-cte-membership-subqueries`.
Scope: X24 only — remaining production SQLite `IN cte` membership
expressions and focused regressions. First inventory current upstream and
reviews, record refreshed base `B`, and bound file ownership here before
editing production code. Then replace shorthand membership with explicit
`IN (SELECT ... FROM cte)` without changing behavior; validate old/new SQL
with focused tests, package race tests and formatting/lint as appropriate.
Do not dispatch X24 concurrently. No existing owner's checkout will be
modified; other independent candidates remain available.

Worktree created from refreshed upstream/4.1 base `B` =
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48`; `_deps` links to `../juju/_deps`.
Inventory confirms eleven production expressions in eight files. Assigned
files (and their existing test files) are:
- `domain/application/state/{application_endpoint,k8sservice}.go`
- `domain/removal/state/model/{application,unit,storage}.go`
- `domain/operation/state/{task,prune}.go`
- `domain/relation/state/relation.go`

No equivalent open/merged upstream review found. #23693 touches `prune.go`
for size expressions, not CTE membership; keep this independent and preserve
that review's separate ownership (possible same-file integration conflict).
Contracts: preserve refresh endpoint filtering, orphan/shared hostname and
pod-device deletion isolation, constraint deletion scope, storage lifecycle
guards, operation completion/size filtering and departed-unit membership.
No schema/generated files, service/API behavior or transaction changes.
Local implementation commit: `8ce0410f515a9b46bc09ba006c108852267eb8e7`.
All eleven expressions use explicit single-column membership subqueries;
existing CTEs, joins, lifecycle guards and transaction behavior are preserved.
Added `TestGetConsumerRelationUnitsChangeDepartureScope` for empty/partial/full
membership and cross-relation/application isolation. The worktree was clean
and was removed on 2026-11-04 after the commit was packaged for transfer.

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

PR description recovered from the original session into
[PR-4.1-cte-membership-subqueries.md](PR-4.1-cte-membership-subqueries.md).
The committed branch is published and linked above; no upstream PR yet.
Opening the PR requires explicit approval. Not landed or PoC-reconciled. No PostgreSQL tests,
full repository suite or live deployment/migration QA were run.

## Sequence review follow-up — X3/X4

**Published / review follow-up in progress (2026-10-30):**
Copilot session `9a4e0037-1cb5-464c-9f9e-d07769103e2b` owns branch
`4.1-sequence-upsert-target` and worktree
`/home/jameinel/dev/go/src/github.com/juju/4.1-sequence-upsert-target`.
Base `B`: `e22f5666d0861d3c84e8465c1ff86fbd23f58f48` (refreshed upstream/4.1).
Published commit: `cfa867609013e22c6ec36a215be50ff80e13c397`.
Verified upstream PR: juju/juju#23731, targeting `4.1`.
Validated local review follow-up commit:
`74aabf93b4ec4337caf5344fb553e9e33c3b5f38`. At that checkpoint the
worktree was clean. This follow-up was published on 2026-11-04 to the
existing branch/PR linked above; no review response was posted.
Do not duplicate this implementation.
Other X3/X4 operations remain available for independent scoping.

Scope: `domain/sequence/state/{sequence.go,sequence_test.go,state_test.go}`
only. `NextValue` and `NextNValues` now target namespace conflicts and
qualify the stored value as `sequence.value`. PostgreSQL requires the
DO UPDATE target and an unambiguous stored-value expression; SQLite/Dqlite
allocation behaviour remains unchanged. No schema/backend-specific changes.
Regressions cover mixed single/batch allocation, static/prefix/empty
namespaces, ordered batches, zero-count no-ops, rollback of new/existing
rows for both allocators, competing single/batch allocations and imported
zero/non-zero sequence continuation through export.

Validation with shared `_deps` and stock SQLite/Dqlite fixtures:

- Before and after the SQL change: `make run-go-tests
  TEST_PACKAGES=./domain/sequence/state
  TEST_FILTER='"TestSequenceSuite|TestStateSuite/TestImportedSequencesContinue"'
  TEST_ARGS='-p 2 -race'`: passed, build tags `dqlite,libsqlite3`.
- `make run-go-tests TEST_PACKAGES='./domain/sequence/...'
  TEST_ARGS='-p 2 -race'`: all sequence packages passed.
- `make pre-check`: passed; its default skips static analysis.
- `RUN_SUBTEST=run_go make static-analysis
  STATIC_ANALYSIS_JOB=test_static_analysis_go`: passed.
- `make run-go-tests TEST_PACKAGES=./domain/sequence/state
  TEST_ARGS='-p 2 -race -c -o <session-binary>'`: passed.
- `timeout 31 stress -p 2 -timeout 30s <session-binary>
  -test.run 'TestSequenceSuite|TestStateSuite/TestImportedSequencesContinue'`:
  10 completed runs, zero failures; two active at the last report. Outer
  timeout ended the bounded run with status 124. Binary removed.
- `gofmt`, prescribed `gci` ordering and `git diff --check`: passed.
- IDE build unavailable because the dedicated worktree is not an open IDE
  project; standard package/test-binary compilation passed instead.

Not run: PostgreSQL execution, full repository suites or controller
bootstrap/migration integration QA. The original session's local PR draft
is preserved in
[PR-4.1-sequence-upsert-target.md](PR-4.1-sequence-upsert-target.md).
Its creation metadata is historical: the PR already exists.
Review follow-up agreed with the user: retain both allocator entry points,
comment on the upsert's first-use/rollback rationale and fixture limitations,
and send concurrent-test results only after the entire transaction succeeds.
**Additional refactor in progress, explicitly requested by the user:**
extract a common private scalar allocator for `NextValue`/`NextNValues` and
use `ON CONFLICT ... RETURNING` to obtain the stored value without a separate
SELECT. Preserve zero-count no-op, namespace isolation, allocation ordering,
rollback and failure detection. This remains owned by the same session and
branch; no additional paths are claimed. Follow-up validation passed: full state suite via
`make run-go-tests TEST_PACKAGES=./domain/sequence/state TEST_ARGS='-p 2 -race'`,
`make pre-check`, `RUN_SUBTEST=run_go make static-analysis
STATIC_ANALYSIS_JOB=test_static_analysis_go`, and test-binary compilation via
`make run-go-tests TEST_PACKAGES=./domain/sequence/state
TEST_ARGS='-p 2 -race -c -o <session-binary>'` (tags `dqlite,libsqlite3`).
`timeout 31 stress -p 2 -timeout 30s <session-binary>
-test.run 'TestSequenceSuite/TestSequenceConcurrentMixedAllocation'`
completed 36 runs with zero failures; two active at the final report, outer
timeout status 124 as intended. Binary removed; formatting/diff checks pass.
No PostgreSQL execution or broader integration QA added.
The original session's unpublished response is preserved in
[COMMENT-4.1-sequence-upsert-review.md](COMMENT-4.1-sequence-upsert-review.md).
It records final refactor lint, compilation and stress as pending; resolve
that evidence gap before making new validation claims or publishing.
Verified on 2026-11-04: the branch points to published
`74aabf93b4`; the common allocator / `RETURNING` refactor and its tests
are uncommitted in `sequence.go` and `sequence_test.go`. The validation
above is historical session evidence, not a fresh check of that dirty
worktree. The older green CI snapshot at `cfa8676090` does not cover the
published follow-up or the uncommitted refactor; assess new-head CI.
Remaining: validate the refactor and obtain approval before publishing it
or posting the exact response; complete upstream review/CI, landing and
PoC reconciliation.

## Candidates needing bounded scope before dispatch

These register findings are not ready-made assignments. Identify remaining
production paths, behaviour contracts and focused tests before branching;
split conflicts by operation and preserve retries, idempotency and errors.

| Finding | Candidate scope / next action |
|---|---|
| X7 | Dispatched as `4.1-drop-iif` above. |
| X24 | Implemented locally by session `767870f3-0997-48f4-a415-c1efab02d088` as `4.1-cte-membership-subqueries`; see its validation/publication status above. Do not duplicate. |
| X3/X4, other upserts | Sequence allocation is implemented in #23731. Inventory any other remaining operations before dispatch; do not recreate that fix or combine unrelated conflict paths. |
| U11 watcher follow-up | Per-offer-connection remote watcher scoping is separate from #23688's offered-application scope. Agree the event/isolation contract and tests before dispatch. |
| X33 stored-bytes follow-up | A stored-bytes metric such as `pg_column_size` needs backend-aware state code; #23693's text-size work does not implement it. Scope separately, not as a mechanical rewrite. |
| X16 dbrepl output follow-up | Feedback on juju/juju#23730 suggests extracting the duplicated row scanning and tabwriter output from `changeLog` and `changeStream` in `internal/worker/dbrepl/worker.go` into a shared helper taking the query and arguments. Preserve each command's SQL, output and error handling; check current upstream code and exercise both commands. Keep separate from the latest-row SQL task. |

Not stock-upstream tasks: the PostgreSQL `constraint.allocate_public_ip`
export-type mismatch needs backend-aware generation; the intentional
`TestSetAPIAddressesSharedPeerRollsBack` NOT NULL failure is not a dqlite
bug. Keep both recorded rather than reopening them as upstream fixes.

Separate documentation task: `4.1-pr-description-guidance` has a staged
[patch](4.1-pr-description-guidance.patch) and
[PR description](PR-4.1-pr-description-guidance.md), but no branch/PR.
Review the description/comment distinction independently of SQL changes.

## PoC reconciliation handoff

Do not implement PoC integration in stock-upstream worktrees. After landing,
hand merged revisions to the integration owner using UPSTREAM-INTEGRATION.md
and update the register. Explicit outstanding obligations: both U15 fixes,
U16's agent-binary fix, U17 guard integration, reconciliation of landed
#23720 and #23726 against
their existing PoC implementations, independent reconciliation of #23688
after it lands, and #23694's UUID-conflict sentinel (not content already
stored). Keep all other landed-fix reconciliation status in the register;
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
