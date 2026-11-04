# Stock upstream Juju 4.1 remaining work

Use this standalone queue for a stock upstream 4.1 checkout. Read the shared
rules, assigned task and target repository's AGENTS.md/contribution guidance;
no PoC checkout, configuration or test setup is required.

This is the recorded 2026-10-30 status snapshot. Check upstream code, GitHub
reviews and CI before acting; implementation, landing and PoC reintegration
are different states. Keep this document focused on pending actions.
[UPSTREAM-DONE.md](UPSTREAM-DONE.md) records implemented work and rationale;
[UPSTREAM-FIXES.md](UPSTREAM-FIXES.md) owns finding/provenance/status mappings.
Neither is prerequisite reading for an assigned task.

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

## Open reviews: finish on existing branches

Do not recreate these implementations. Verify outstanding review requests,
CI and mergeability; validate any follow-up against the current head.

| Branch / PR | Finding | Remaining action / acceptance contract |
|---|---|---|
| `4.1-model-config-precedence`, #23683 | U7 | Complete review/CI. `v_model_config` and `domain/modelconfig/state` must exclude competing agent stream/version rows; test config values sorting before and after authoritative values and one effective value per key. |
| `4.1-crossmodel-secret-revisions`, #23688 | U11 | Complete remote-query review/CI in `domain/crossmodelrelation/state/model/secrets.go`. Report any-unit-behind secrets, latest revision once, via indexed offer connections. Cover no/current/behind/mixed consumers, multiple secrets/revisions, prefix collisions and filtering; assert revisions and no duplicates. Keep schema/import/recovery/cleanup together. |
| `4.1-operation-query-portability`, #23693 | X34/X33 | Complete review/CI in `domain/operation/state`; keep boolean comparisons and text-size expressions in separate commits. Verify filtering/pruning thresholds across value types. |
| `4.1-nullable-primary-keys`, #23710 | U15, charm | Complete review/CI and land charm first. Required container key, manifest-position key and duplicate-position import diagnostics/tests are implemented; do not redo them. |
| `4.1-application-exposure-scope`, #23723 | U15, exposure | Review against charm, then rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors. Table-specific errors are implemented; assess new-head CI. |
| `4.1-local-secret-revisions`, #23720 | U11 follow-up | Add review regressions in `domain/secret/state/state_test.go`: multiple behind secrets for one unit, empty/nonexistent units, initial-query -> changes-query round trip and explicit nil results. Exercise both `InitialWatchStatementForConsumedSecretsChange` and `GetConsumedSecretURIsWithChanges`; tests must pass with old and portable SQL. |
| `4.1-lookup-key-not-null`, #23724 | U17 | Expand nullable payload validation and factor typed lookup deltas; use the focused checklist below. Independent of both U15 branches. |
| `4.1-dbaccessor-pool-cleanup`, juju/juju#23726 | U14 | Complete review/CI on `ee55bab5f4`, independently based on `e22f5666d0`. Startup/replacement cleanup and nine regressions are implemented; preserve retained-pool ownership and successful transfer. Review is required and CI is still running; do not recreate the implementation. |
| `4.1-dbrepl-latest-row`, juju/juju#23730 | X16, separate REPL query | Complete review/CI on `49a1bdf808`, independently based on `e22f5666d0`. Preserve `.change-stream` fields, per-namespace/key latest IDs, `id > 0` and ordering. Four regressions pass with old/new SQL on SQLite/Dqlite; do not recreate the implementation. No replication state is involved. |

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

### `4.1-agentbinary-conflict` — U16 / X18

**Implemented locally / awaiting review (2026-10-30):** Copilot session
`2b06495a-18f9-4418-ba5c-b2e8fd908dd1` owns branch
`4.1-agentbinary-conflict` and worktree
`/home/jameinel/dev/go/src/github.com/juju/4.1-agentbinary-conflict`.
Refreshed upstream 4.1 base: `e22f5666d0861d3c84e8465c1ff86fbd23f58f48`.
Published implementation: `323807680f7a5249f65d2d7378c0c2158cb3edee`.
Comment-only follow-up `6a66e5ed1be4cc0bde5fee269b6da1f87da56440` describes
registration/immutability rather than naming a SQL backend; committed and
pushed after approval. Worktree clean; formatting/diff checks passed.
Tests not rerun for the comment-only follow-up.
Pushed to `jameinel/juju:4.1-agentbinary-conflict`; approved PR published:
[juju/juju#23728](https://github.com/juju/juju/pull/23728), targeting `4.1`.
Approved revised description published and verified against
`PR-4.1-agentbinary-conflict.md`; it focuses on the operation and its
rationale. Scope is
`domain/agentbinary/state/{controller,model}` only; no other checkout owned.
Plan completed: checked upstream/reviews, added targeted conflict-safe
inserts and affected-row checks, preserved error sentinels and immutability,
and tested same-transaction writes and rollback for both conflict cases.
Validation used shared `_deps` and stock dqlite fixtures only:

- `make run-go-tests` with
  `TEST_PACKAGES='./domain/agentbinary/state/controller ./domain/agentbinary/state/model'`,
  `TEST_FILTER='Test.*StateSuite/TestAdd'`, `TEST_ARGS='-p 2 -race'`: passed.
- Same command without `TEST_FILTER`: both full state race suites passed.
- Each package compiled with `make run-go-tests`,
  `TEST_ARGS='-p 2 -race -c -o <session-binary>'`; then
  `timeout 31 stress -p 2 -timeout 30s <session-binary> -test.run <suite/TestAdd>`:
  controller 30/model 20 completed iterations, zero reported failures;
  bounded runs ended at the outer timeout. Binaries removed afterwards.
- `make pre-check`: passed; its default skips static analysis.
- `RUN_SUBTEST=run_go make static-analysis STATIC_ANALYSIS_JOB=test_static_analysis_go`:
  Go lint passed. Other static-analysis jobs/full repository tests not run.
- `gofmt`, prescribed `gci` ordering, and `git diff --check`: passed.

Remaining: upstream review/CI and landing; not landed or PoC-reconciled.
Do not duplicate this branch; consult later tasks for current assignments.

Both `domain/agentbinary/state/{controller,model}/state.go`
`RegisterAgentBinary` paths SELECT after a primary-key error, which aborts
a PostgreSQL transaction. Use `ON CONFLICT DO NOTHING`, then compare,
preserving `AlreadyExists` / `AgentBinaryImmutable`, retries and idempotency.
Test new, repeated and different-object registrations in both state suites;
verify the conflict path leaves the transaction usable.

### `4.1-lease-pin-conflict` — X18

**Claimed / in progress (2026-10-30):** Copilot session
`a646e393-debb-413a-851c-ca78d13c9a23` owns branch
`4.1-lease-pin-conflict` and worktree
`/home/jameinel/dev/go/src/github.com/juju/4.1-lease-pin-conflict`.
Base `B` = `e22f5666d0` (upstream/4.1, PR #23722 merge); no existing
upstream PR found. Scope is `domain/lease/state` only. Plan:
check upstream code/PRs for existing work, replace unique-violation
handling in `PinLease` with `ON CONFLICT DO NOTHING`, add competing-pin and
transaction-usability regressions, run focused then package validation.
Other sessions may take `4.1-dbrepl-latest-row` or later independent
tasks; do not duplicate this claim.
Status: `373ab4f422` pushed to `origin/4.1-lease-pin-conflict`; opened
juju/juju#23729 from [PR-4.1-lease-pin-conflict.md](PR-4.1-lease-pin-conflict.md).
Awaiting review/CI. Lint clean. Tests:
`make run-go-tests TEST_PACKAGES=./domain/lease/state/...` with
`-race -count=3` (pass; also pass on old SQL, as SQLite keeps the txn
usable) and `-race -count=20 -run TestStateSuite/TestPin` (pass).

`domain/lease/state` `PinLease` ignores a unique violation. Resolve competing
pins in SQL with `ON CONFLICT DO NOTHING`, preserving the existing
already-pinned result. Test competing pins and continued transaction use.
This is portability, not a current dqlite defect; keep production timing
changes out of this branch.

### `4.1-secret-latest-revision`

After the consumed-secret reviews (#23688/#23720), independently rewrite
five queries in `domain/secret/state/state.go`: `updateSecret`,
`GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and
`getSecretsRevisionExpiryChanges`. Explicitly select the latest revision
instead of relying on ungrouped columns beside MAX. Existing and new SQL
must pass regressions with multiple revisions, expiry/rotation rows and
both owner kinds. This does not duplicate #23720's two local queries.

**Blocked (2026-10-30):** #23688 and #23720 remain open (REVIEW_REQUIRED;
#23688 DIRTY) and touch `domain/secret/state/{state,state_test}.go`.
Not claimed; X7 was dispatched instead.

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
the CASE version `6b5aa7348d`) is local only:
not pushed and no PR. No upstream PR was found. Schema-view commit in
progress.
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
Remaining: approval to push to `jameinel/juju` and open against `4.1`.

## Lookup-key NOT NULL task (separate session)

Continue existing independent #23724, not a new implementation branch.
Scope: single-column `id INT PRIMARY KEY NOT NULL` (48 model, 17 controller
at `8369e6e2a9`), generated scalar IDs and consumed payload boundaries.
Recheck inventory at the actual base; leave INTEGER rowid keys, composite
keys, historical DDL, 4.0.12 export types and unrelated nullable fields alone.
Do not replace explicit schema constraints with generator inference.

Remaining review/acceptance checklist:

- Factor a small typed-delta helper; exercise every consumed legacy lookup
  type's full row, multiple rows, nullable non-ID fields, nil IDs and valid
  zero. Reject missing IDs with table/row context and invalid-payload
  classification; never invent zero. No legacy controller delta is needed.
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

## CTE membership subqueries — X24

**Implemented locally / awaiting publication approval (2026-10-30):** Copilot session
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
membership and cross-relation/application isolation. Worktree clean.

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

PR description prepared locally in this session's
`files/PR-4.1-cte-membership-subqueries.md`. No push or PR yet; publication
requires explicit approval. Not landed or PoC-reconciled. No PostgreSQL tests,
full repository suite or live deployment/migration QA were run.

## Sequence allocation upsert — X3/X4

**Published / review follow-up in progress (2026-10-30):**
Copilot session `9a4e0037-1cb5-464c-9f9e-d07769103e2b` owns branch
`4.1-sequence-upsert-target` and worktree
`/home/jameinel/dev/go/src/github.com/juju/4.1-sequence-upsert-target`.
Base `B`: `e22f5666d0861d3c84e8465c1ff86fbd23f58f48` (refreshed upstream/4.1).
Published commit: `cfa867609013e22c6ec36a215be50ff80e13c397`.
Verified upstream PR: juju/juju#23731, targeting `4.1`.
Validated local review follow-up commit:
`74aabf93b4ec4337caf5344fb553e9e33c3b5f38`. Worktree clean; this follow-up
has not been pushed or commented publicly. Do not duplicate this implementation.
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
bootstrap/migration integration QA. Local PR draft with exact destination,
proposed title/body and validation is retained at
`/home/jameinel/.copilot/session-state/9a4e0037-1cb5-464c-9f9e-d07769103e2b/files/PR-4.1-sequence-upsert-target.txt`.
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
Draft response is local at the same session's `files/review-response-23731.txt`.
Remaining: obtain approval before pushing the follow-up
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
| X3/X4 | Sequence allocation implemented locally as `4.1-sequence-upsert-target`; see validation and ownership below. Other operations remain unclaimed; keep distinct from landed objectstore and queued agentbinary/lease work. |
| U11 watcher follow-up | Per-offer-connection remote watcher scoping is separate from #23688's offered-application scope. Agree the event/isolation contract and tests before dispatch. |
| X33 stored-bytes follow-up | A stored-bytes metric such as `pg_column_size` needs backend-aware state code; #23693's text-size work does not implement it. Scope separately, not as a mechanical rewrite. |

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
U17 guard integration, independent reconciliation of #23688/#23720 despite
existing PoC cherry-picks, and #23694's UUID-conflict sentinel (not content
already stored). After U14 #23726 lands, compare its final cleanup/tests with
PoC `b6a7690d80` and retain the PostgreSQL tracker/backend adaptations; the
open upstream PR is not yet reconciled. Keep other landed-fix status in the
register; upstream landing alone does not mark it complete.

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
