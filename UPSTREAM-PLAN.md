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

The open-review snapshot below is from 2026-11-09 18:56 UTC. Recheck heads,
reviews and CI before acting; implemented scope and historical validation are
in UPSTREAM-DONE.md. Skipped checks are not passes, and publication does not
establish QA acceptance.

## Purpose and SQL/schema acceptance

These breakouts bring SQL and schema improvements identified by the PostgreSQL
PoC into stock upstream 4.1. The deliverable is correct, explicit SQL and
appropriately typed schemas running on SQLite and dqlite, not PostgreSQL
support. Do not introduce PostgreSQL-specific SQL, backend switching, runtime
dependencies or a PostgreSQL test-harness requirement into these patches.
Use portable SQL where it expresses the intended behavior on both supported
backends; PostgreSQL can help expose ambiguity, but does not define Juju's
domain semantics.

Fix the SQL/schema root cause, not just the Go conversion boundary. A
successful bind, scan or round trip does not establish that the stored type,
query semantics or SQL formulation is correct. Do not preserve an incorrectly
typed column by adding Go formatting/parsing, or hide a type mismatch with a
cast solely to make the current representation work. For example, U21 stores
integer relation IDs as integers; legacy text-key conversion belongs in
import, not in every normal read and write.

Boundary conversions remain appropriate for genuine legacy import/export or
external API compatibility. Documented driver-defect workarounds, such as the
temporary nullable-BOOLEAN casts below, have a separate rationale and removal
condition; they are not precedent for masking schema defects. This goal does
not turn explicitly scoped tooling, decoding or driver fixes into SQL rewrites.

Before implementing, check that the task's prescribed approach addresses the
underlying SQL/schema issue. PoC implementations are evidence, not mandatory
designs. If a task says to preserve a representation or avoid schema changes
but that appears to leave the root problem intact, stop and ask the user.
Do not silently follow a conflicting prescription or expand the breakout.
Schema changes still require the release-policy and compatibility checks
below; this goal does not waive them.

Acceptance requires explaining how the actual SQL/schema change removes the
identified issue, as well as validating behavior on SQLite and real dqlite.
Passing those tests alone does not demonstrate portable SQL: both can accept
the same permissive construct. PostgreSQL execution is optional and available
to resolve uncertain SQL/schema semantics; ask the user when it is needed.
Report it separately and never claim PostgreSQL acceptance or execution from
SQLite/dqlite results.

## D55 implementation review (2026-11-06)

Reviewed implementation SQL/schema hunks and relevant conversion boundaries
against upstream `148e09f015` and the GitHub PR heads, including U21
`2f680f1302`, sequence `696c116f79` and the local grouped-column commit
`c3f75cf032`. This is a design-alignment review, not fresh test, CI or
functional-QA acceptance. Existing validation remains revision-specific.

### Rework and merge holds

| Branch | Disposition / required next action |
|---|---|
| `4.1-storage-filesystem-attachment-row`, #23766 | **Hold merge; plan rework.** Lowest attachment UUID makes the result deterministic, but does not establish the correct mount point. The user confirmed that the filesystem/node attachment invariant is uncertain. Investigate creation, sharing, detach/reattach, life transitions and import/recovery before choosing a design. If duplicates are invalid, scope schema enforcement and explicit handling of legacy duplicates; if valid, establish the domain-based row-selection contract. Do not silently add uniqueness, discard rows or accept UUID ordering as the final semantic fix. |
| `4.1-unitstate-relation-key`, #23769 | The original text-binding implementation required rework; **the schema-first rework is implemented** at `2f680f1302`. Review the net patch, integer export/import/recovery and relation deletion paths. Do not revive the superseded normal-path conversions or classify the old PoC workaround as the completed fix. |
| `4.1-x16-grouped-columns` | The original portability-only scope required rework because the row converters still discarded CA certificates. **Rework is pushed** at `c3f75cf032`: corrected SQL retains the rows and both consumers collect all certificates. Validate the final multi-certificate/attribute/auth-type contract before publication; do not revert to a syntax-only patch. |

These are not instructions to rewrite every open branch. Model-config
#23683, remote-secret #23688, operation-expression #23693, charm #23710,
exposure #23723, lookup #23724, lease #23729, sequence #23731, membership
#23750 and IIF #23751 already address their scoped SQL/schema concern.
Continue their review and required QA; no D55-driven redesign was identified.
For #23693, measuring explicit text representations is intentional; a
physical stored-bytes metric would be a different contract, not a reason to
retype correct integer/Boolean/timestamp columns.

Keymanager #23756, nullable-Boolean #23759 and controller-export #23765
are justified, bounded driver workarounds, not incorrect-schema preservation.
Keep their removal/review conditions and follow the dependency fixes; do
not replace correct BOOLEAN/timestamp storage with integer/text storage.
The dbrepl output refactor and PR guidance are separate, non-SQL concerns.

### Merged work and reasonable follow-ups

The [merged implementation assessment](UPSTREAM-DONE.md#d55-merged-implementation-assessment-2026-11-06)
records why the landed schema, query and representation fixes align.
Do not reopen them merely because they contain Go conversions.

- U13/#23684 needs the already-open #23756 follow-up for dqlite's
  `RETURNING` error-reporting defect. Preserve the unknown-algorithm guard
  and correct inserted-ID behavior; never restore `LastInsertId`.
- U4/#23686 fixed setup error checking but still seeds through SQLite
  `datetime('now', ?)` expressions. An independent, low-priority fixture
  portability follow-up is [queued below](#todo-lease-migration-fixture-timestamps-low-priority):
  compute offsets using existing lease
  timestamp conventions and bind the values. Preserve held/expired,
  model/type isolation and setup-error assertions; validate SQLite and real
  dqlite. No lease schema change or PostgreSQL support belongs in that patch.
  The earlier claim that the merged patch already bound timestamps was
  inaccurate; it described the PoC approach.
- The remaining latest-secret queries (U19) and unresolved per-connection
  watcher contract remain separately scoped work, not reasons to redo
  merged #23720's local-consumer SQL.

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
- `go` is a snap on this host. Never redirect its output, or the output
  of `make` targets that run it, straight to a file: AppArmor silently
  discards it, and the exit code is unaffected. Pipe instead, for example
  `set -o pipefail; go test ... 2>&1 | tee "$LOG"`.
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

## Mainline integration requests

1. Request a merge of upstream `4.1` into `main` and check that the
   integration is clean, including conflicts and appropriate validation.
   Check current branch heads and obtain approval before publishing or
   merging anything.
2. Afterward, audit `main` for SQL `IIF` and ensure it is cleaned up.
   [juju/juju#23751](https://github.com/juju/juju/pull/23751) targets `4.1`
   and is still in review; verify whether its changes reached `main`
   rather than assuming the merge included them. If `IIF` remains,
   prepare a separately scoped mainline change preserving its behavior.

### Local mainline merge preparation (2026-11-09)

Worktree `4.1-into-4.2`, branch `4.1-into-4.2`: local merge commit
`496c7d78187be3d0425fb66b386e17638872ce4a` merges upstream `4.1`
`bd36cef53587f60bbdb4d910eb6338b9d95fba4a` into upstream `main`
`419d02804e5df3a1facb80c7eb749be7176bfc84`. Git reported no conflicts;
the merge tree matches the independent `git merge-tree` result. It brings
14 commits, including three PR merges. The branch is pushed to
`jameinel/juju:4.1-into-4.2` and proposed as
[#23775](https://github.com/juju/juju/pull/23775), targeting `main`; the
integration has not landed. The PR body is saved at
[PR-4.1-into-4.2.md](PR-4.1-into-4.2.md).

- Affected backups, CLI commands, modelcmd, agent-binary controller/model
  state and resource state suites passed with `-p 2 -race -count=1` on
  SQLite and with `make run-go-tests` using real dqlite and the shared
  `_deps` headers and static libraries.
- `make go-build`, `STATIC_ANALYSIS=1 make pre-check` and
  `git diff --check` passed. Live controller and migration QA were not run.
  The first pre-check attempt with `GOPROXY=off` failed on an uncached
  license-analysis dependency; the normal-proxy retry passed.
- The merge still contains four SQL `IIF` expressions. #23751 remains open
  against `4.1`, so its cleanup is not part of this merge. Audit `main`
  again after integration rather than treating this inventory as final.
- Existing `.github/workflows/merge.yml` checks `4.1` -> `main` for
  conflicts but does not publish or merge PRs. Automated merge-forward PRs
  are proposed in [#23709](https://github.com/juju/juju/pull/23709), which
  is still open. No upstream `4.2` branch existed when checked.

At 2026-11-09 18:56 UTC, upstream `main` remained at `419d02804e`, but
`4.1` had advanced to `775513d9df` with merged #23757, two commits beyond
the source in this proposal. No human review had been submitted; GitHub
required review. The [localhost LXD smoke](https://github.com/juju/juju/actions/runs/42647771546/job/129265668097)
and aggregate failed after cleanup, with no established cause.

Next: investigate CI, refresh the merge against current upstream heads,
complete live controller/migration QA, and obtain explicit approval before
merging. Prior validation applies to `496c7d7818`, not a refreshed merge.

## Breakout claims

This table is the authoritative local claim list for new breakouts. Record
the task, owner/session, branch, worktree, base, status and next action.
Keep blocked and paused work claimed; do not assume an existing worktree is
available. Use `reserved` until the worktree and base are confirmed, then
`in progress`, `blocked`, or `published` as appropriate.

| Task | Owner/session | Branch | Worktree | Base | Status | Next action |
|---|---|---|---|---|---|---|
| U22 filesystem attachment row | `fc0b395c-35d9-4277-ba1d-3bc4779db286` | `4.1-storage-filesystem-attachment-row` | `juju-4.1-storage-filesystem-attachment-row` | `148e09f015fe59237f382b9df1a468c15ee7fa42` | blocked | [#23766](https://github.com/juju/juju/pull/23766), head `65dbcae6a9c3f117948ef667c43d195ad8a38b09`; user-requested local merge hold under D55. Investigate the attachment invariant and plan semantic rework before merge; retain ownership. Earlier SQLite/dqlite race, lint and txncheck passes do not settle the correct-mount contract. Live QA remains pending. |
| U21 unitstate relation key | `750b409e-35d5-4196-9b64-56ee36a49f56` | `4.1-unitstate-relation-key` | `juju-4.1-unitstate-relation-key` | `148e09f015fe59237f382b9df1a468c15ee7fa42` | published | [#23769](https://github.com/juju/juju/pull/23769), head `9a3083a887952914d1457d10084bacfa78bd0c6d`; bot feedback answered, human review required and CI running. Forward-port to main after landing and check migration/export compatibility; reconcile the PoC workaround. The user reports performing functional QA while editing the PR; individual outcomes were not independently verified. At `2f680f1302`, SQLite affected race suites passed; dqlite suites passed except two known #23759 nullable-Boolean machine failures, with changed machine tests passing separately. Scoped lint, txncheck and export regeneration passed. The comment-only follow-up passed transformer package tests, scoped lint, txncheck and gci; default pre-check skipped static analysis. PostgreSQL not tested. |
| X16 secret backend grouped columns | `4d032417-af51-4f77-b950-bf70dd707e46` | `4.1-x16-grouped-columns` | `juju-4.1-x16-grouped-columns` | `148e09f015fe59237f382b9df1a468c15ee7fa42` | in progress | Pushed commit `c3f75cf03262a98d3d1ed0606732b502a4b673e7` preserves all CA certificates in both consumers. SQLite/dqlite race suites, scoped lint, txncheck and default pre-check passed (static analysis skipped there); live QA and PostgreSQL execution not run. Proposed PR: `PR-4.1-x16-grouped-columns.md`, base `juju/juju:4.1`, head `jameinel:4.1-x16-grouped-columns`. QA steps are unverified and require review; do not create the PR without explicit approval. |

## Open reviews: finish on existing branches

Do not recreate these implementations. Verify outstanding review requests,
CI and mergeability; validate any follow-up against the current head.
Apply the D55 rework/merge holds above before treating a branch as ready.

At 2026-11-09 18:56 UTC all 17 queued PRs, including #23766 and #23775
outside this table, were open and GitHub required review. None was ready to
merge. Current macOS and aggregate checks passed on #23688, #23693 and
#23710, superseding their earlier cancelled-job failures. Skipped checks
remain unvalidated; review approval, QA and stack prerequisites are separate.

| Branch / PR | Finding | Remaining action / acceptance contract |
|---|---|---|
| `4.1-model-config-precedence`, [#23683](https://github.com/juju/juju/pull/23683) | U7 | Review follow-ups are published; reported checks have no failures or pending jobs. Obtain remaining approval. Preserve authoritative agent-table values, import filtering and one effective value per key. |
| `4.1-crossmodel-secret-revisions`, [#23688](https://github.com/juju/juju/pull/23688) | U11 | Await requested human review; bot findings were answered. Keep any-unit-behind semantics, indexed connections, schema/import/recovery/cleanup and regressions together; reconcile the PoC after landing. |
| `4.1-operation-query-portability`, [#23693](https://github.com/juju/juju/pull/23693) | X34/X33 | Human approval exists; await the remaining GitHub review gate. Commit identity/message and rebase requests were addressed. Preserve filtering and the reviewed NULL-as-zero column-size fix, which changes the old whole-row-ignored estimate. |
| `4.1-nullable-primary-keys`, [#23710](https://github.com/juju/juju/pull/23710) | U15, charm | Simon approved; await the remaining GitHub review gate and land charm before exposure. Required container key, manifest-position key and duplicate-position diagnostics/tests are implemented. |
| `4.1-application-exposure-scope`, [#23723](https://github.com/juju/juju/pull/23723) | U15, exposure | Reported checks have no failures or pending jobs; obtain approval. Rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors. |
| `4.1-lookup-key-not-null`, [#23724](https://github.com/juju/juju/pull/23724) | U17 | Reported checks have no failures or pending jobs; obtain approval. Schema-derived decoder inventory, direct `NotValid` classification and the transaction-test fix are implemented. Retain explicit typed deltas and the 65-key scope; later reconcile the combined guard with exposure. |
| `4.1-lease-pin-conflict`, [#23729](https://github.com/juju/juju/pull/23729) | X18 | Simon approved on November 9 and suggested removing `database.IsErr*` methods in favor of queries up front; evaluate and respond or defer. GitHub still requires review. Preserve competing-pin, repeated-pin and missing-lease behaviour. |
| `4.1-sequence-upsert-target`, [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | Head `696c116f79` replaces RETURNING with a checked upsert and same-transaction lookup to preserve retryable errors. Current reported checks have no failures or pending jobs; Simon's approval predates this head. Complete functional QA and review. Preserve zero-based allocation, contiguous batches, namespace isolation, rollback and import continuity. |
| `4.1-drop-iif`, [#23751](https://github.com/juju/juju/pull/23751) | X7 | Head `fcec930751`: investigate failed revision and charms smoke, upgrade pre-check and aggregate; obtain human review. Complete migration QA and reconcile with the PoC after landing. Preserve the equivalent null-check flags in the connection query and controller views. |
| `4.1-cte-membership-subqueries`, [#23750](https://github.com/juju/juju/pull/23750) | X24 | Head `67daa565a6`: gfouillet approved; his follow-ups were implemented and answered, awaiting confirmation. Investigate revision smoke and aggregate failures; validate the follow-up on real dqlite and complete manual QA. Preserve explicit single-column membership and scoped cleanup, lifecycle and departure semantics. |
| `4.1-pr-description-guidance`, [#23755](https://github.com/juju/juju/pull/23755) | Agent guidance | Head `b78a42c7c9`: await Simon's response to the November 9 reply about moving guidance into `AGENTS.md`. Seven checks passed and 12 were skipped. Keep reviewer context, prospective merge-message guidance and reproducible reviewer QA distinct from review history. |
| `4.1-lxd-default-bases`, [#23756](https://github.com/juju/juju/pull/23756) | Integration-test diagnostics and dqlite CI | Head `63a93201b5` includes the keymanager fix, dqlite RETURNING sentinel and SQLite WAL-mode assertion. Human approval predates this head; await the remaining review gate. Preserve SSH-key diagnostics and failure status under `set -e`. |
| `4.1-dqlite-nullable-boolean`, [#23759](https://github.com/juju/juju/pull/23759) | Real-dqlite domain CI | Head `a848d4426e` is based on #23756 at `63a93201b5`; await review and rebase onto `4.1` after the parent lands. Current Unit Tests, upgrade pre-check and aggregate passed. Preserve NULL/false/true and the upstream-fix sentinel. |
| `4.1-controller-export-null`, [#23765](https://github.com/juju/juju/pull/23765) | Controller backup NULL fidelity | Independent head `2e12c3150a`. Evaluate Van Cleef's three suggestions; investigate failed localhost smoke and aggregate, obtain human review and complete disposable-controller backup QA. Preserve nullable Boolean and timestamp values without changing the export format. |
| `4.1-unitstate-relation-key`, [#23769](https://github.com/juju/juju/pull/23769) | U21/X23 | Head `9a3083a887`, based on `148e09f015`. Van Cleef's suggestions were [answered](https://github.com/juju/juju/pull/23769#issuecomment-6756558330): the transform comment explains the conversion without claiming canonical-only inputs; mainline propagation is deferred until after landing. Obtain human review and monitor CI. The user reports performing QA, but individual results were not independently verified. Preserve integer keys, legacy conversion and orphan checkpoints without a new foreign key; check migration/export compatibility when forward-porting and reconcile the different PoC workaround after landing. |

Review and CI evidence at that snapshot:

- #23766: Van Cleef's November 9 verdict was Comment, with one issue and
  two suggestions. Evaluate its claim that UUID selection may pick a stale
  attachment; the D55 merge hold and invariant investigation remain.
- #23765: Van Cleef's November 9 Comment had no issues and three suggestions:
  broader nullable-field coverage, generator-drift checking and CI triage.
  #23769 had no issues and two suggestions: mainline propagation and the
  legacy-key conversion rationale. Subsequently, both were answered in
  #23769; the rationale was documented and forward-porting was deferred.
  These findings are not automatic merge blockers.
- The [#23750 revision](https://github.com/juju/juju/actions/runs/42653392922/job/129285455048),
  #23751 [revision](https://github.com/juju/juju/actions/runs/42636373516/job/129225558822)
  and [charms](https://github.com/juju/juju/actions/runs/42636373516/job/129225558842),
  and [#23765 localhost](https://github.com/juju/juju/actions/runs/42258592467/job/128230348819)
  LXD smoke jobs failed after cleanup; their aggregates also failed. Do not
  infer a shared cause. #23751's separate
  [upgrade pre-check](https://github.com/juju/juju/actions/runs/42636373516/job/129225558535)
  reported `no snap found for "juju"`.

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

### TODO: lease migration fixture timestamps (low priority)

**Status: pending; unclaimed.** Independent follow-up to merged
[juju/juju#23686](https://github.com/juju/juju/pull/23686) (U4), not a merge
blocker or a reason to reopen that PR.

In `domain/lease/state/migration_test.go`, replace SQLite-specific
`datetime('now', ?)` fixture expressions with computed timestamp values
bound as query parameters. Follow existing lease clock and timestamp
conventions so held and expired fixtures remain consistent with the code
under test.

Preserve setup-error assertions, held/expired lease coverage and model/type
isolation. Validate the affected migration tests on SQLite and real dqlite.
Keep this test-only: no lease schema change, production behavior change or
PostgreSQL runtime/harness dependency.

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

Related but distinct:
[canonical/dqlite#920](https://github.com/canonical/dqlite/pull/920) merged
on 2026-11-09 and preserves busy errors otherwise misreported as a nonempty
statement tail. Merge details are in UPSTREAM-DONE.md. Identify a release or
approved backport and validate Juju's actual dependency before treating this
path as fixed in Juju; the response-buffer fix alone does not resolve it.

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
  review. The later 2026-11-09 snapshot reports successful upgrade pre-check
  and aggregate jobs; continue monitoring the real-dqlite domain step's
  runtime on subsequent heads.
- [ ] Finish #23765 review, CI and functional backup QA independently of
  #23759. Retain the distinct NULL, false, true and timestamp cases.
- [ ] When Juju bundles a release containing #883, review the temporary
  casts, sentinel and model/controller export NULL markers; sweep nullable
  Boolean reads. Verify NULL, false and true remain distinct where required,
  along with nullable timestamps, on real dqlite and SQLite. Restore
  `constraint.allocate_public_ip` to `BOOLEAN` (DIALECT.md X25).

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
on the existing branch, but **hold merge and plan rework**. The user
confirmed that the attachment invariant is uncertain; investigate before
choosing schema enforcement or a domain-based selection rule. Live QA
remains pending, and a passing deterministic-row regression is insufficient.

`domain/storage/state/instance.go` `GetStorageInstanceInfo` joins every
filesystem attachment on the unit's net node and collapses the fan-out
with `GROUP BY sa.uuid`. The schema does not forbid two filesystem
attachments for the same filesystem and net node, so dqlite returns an
arbitrary mount point. The published implementation picks the lowest
attachment UUID via a correlated subquery, drops the `GROUP BY`, and moves
the `ORDER BY` out of the `FROM` subquery in both the attachment and the
block-device link queries. Regression: two filesystem attachments on the
same node; the PoC test failed 7 of 10 runs against the old query.
This is not yet a justified correct-mount contract. Investigate whether
the schema should forbid duplicates (a unique index on
`(storage_filesystem_uuid, net_node_uuid)`) or legitimate lifecycle states
need another selection rule. Any constraint needs writer, import/export
and recovery review, including existing duplicate handling. Do not choose
the invariant or silently deduplicate data without approval. PoC: `640d3bcabf`.

#### `4.1-unitstate-relation-key` — U21, type mismatch

Published as [#23769](https://github.com/juju/juju/pull/23769), head
`9a3083a887`; continue human review and CI on the existing branch. After
landing, propagate the change to main, checking migration/export compatibility.

`setUnitStateRelation` binds Go `int` relation IDs to the `TEXT`
`unit_state_relation.key`, and `GetUnitState` scans it back into `int`.
The user chose to change the column to `INT`, regenerate the 4.1 export
type and convert legacy text keys during import, without adding a foreign
key. Preserve relation filtering and deletion, reject invalid legacy keys,
and validate schema, import/export, recovery and unit-state behavior. The
PoC `1afd3b9434` uses a different, superseded text-binding approach.

#### `4.1-x16-grouped-columns` — U22, portability

Remove ungrouped projections while preserving secret counts and backend
configuration. CA certificates are not functionally dependent on the old
grouping keys: a cloud can have several. Preserve every certificate in both
consumers, deduplicating repeated join rows; this fixes the previous loss of
all but an unspecified certificate.

The original [#17993](https://github.com/juju/juju/pull/17993) addressed
model-specific Kubernetes credentials. The later
[#22027](https://github.com/juju/juju/pull/22027) documents one displayed
backend per model; credential attributes are merged into that backend.
Neither establishes an intent to discard additional cloud CA certificates.

- `domain/secretbackend/state/state.go`: `ListSecretBackends` and the
  in-use Kubernetes backend query group by every selected column;
  `getK8sSecretBackendForModel` uses `SELECT DISTINCT`. PoC `52a0eea6ee`.
- `domain/removal/state/controller/secret.go`
  `getK8sSecretBackendForModel`: `SELECT DISTINCT` instead of
  `GROUP BY vm.name, vcca.attribute_key`; adds
  `TestGetActiveModelSecretBackendK8sBuiltIn`, which had no coverage
  before. PoC `367f754dd6`.

Both `getK8sSecretBackendForModel` copies and the in-use listing must return
all cloud CA certificates, including with multiple credential attributes and
cloud auth types. Validate counts and certificate deduplication on both
SQLite and real dqlite.

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
| [4.1-drop-iif](https://github.com/jameinel/juju/tree/4.1-drop-iif), [juju/juju#23751](https://github.com/juju/juju/pull/23751) | `fcec930751`: migrating-model test follow-up to the query and schema-view changes | Human review, CI triage and functional QA remain. |
| [4.1-cte-membership-subqueries](https://github.com/jameinel/juju/tree/4.1-cte-membership-subqueries), [#23750](https://github.com/juju/juju/pull/23750) | `67daa565a6`: review follow-ups to the eleven membership expressions and departure-scope regression | Reviewer confirmation, real-dqlite validation, CI triage and manual QA remain. |
| [4.1-sequence-upsert-target](https://github.com/jameinel/juju/tree/4.1-sequence-upsert-target), [juju/juju#23731](https://github.com/juju/juju/pull/23731) | [696c116f79](https://github.com/jameinel/juju/commit/696c116f7943d16e6d1706fc857c561f90ee8aa3): checked upsert plus same-transaction lookup and stale-snapshot regression, superseding `76b4a04f09`'s RETURNING implementation | Review and functional QA remain; check CI again before acting. |

Use these published heads. Historical validation below is not a new check
of the latest heads.

### `4.1-drop-iif` — X7

**Published for review (2026-11-04):** [juju/juju#23751](https://github.com/juju/juju/pull/23751)
targets `4.1` from `4.1-drop-iif`, originally with base
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` and head
`adf9f317a6962f5722bb1f4e401db3e9723fd768`; the November 9 head is
`fcec930751`. The query commit
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

Remaining: triage the current-head failures described above, obtain review,
complete functional QA, and reconcile the PoC after landing. The published PR
description is also saved locally at
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

PR [juju/juju#23750](https://github.com/juju/juju/pull/23750) remains open
against `4.1`. On November 9, head `67daa565a6` included and answered
gfouillet's November 6 review requests, though the threads remained open.
SQLite race suites passed for the follow-ups; a real-dqlite run did not build
because `dqlite.h` was missing. The current head's revision smoke and
aggregate failed; human confirmation, real-dqlite validation and manual QA
remain. Earlier passing validation above is for `02e9f73e49`. The PoC retains
earlier equivalent SQL, not a verified reconciliation of the final upstream
result. No PostgreSQL execution or full repository suite was recorded.

## Sequence review follow-up — X3/X4

Continue existing [juju/juju#23731](https://github.com/juju/juju/pull/23731)
at `696c116f7943d16e6d1706fc857c561f90ee8aa3`. The shared scalar allocator
uses an explicitly targeted upsert, checks one affected row, then reads the
value in the same transaction. This replaces the earlier RETURNING version
because dqlite and Sqlair can lose retryable write-conflict errors.
The workaround does not mask a schema mismatch; review it when both error
paths are fixed, not merely when dqlite changes.

Preserve both allocator entry points, zero-count no-op, namespace isolation,
zero-based contiguous allocation, rollback and suppressed-write detection.
Concurrent tests must collect results only after the whole transaction
succeeds. The fixtures exercise SQLite/Dqlite contracts, not PostgreSQL SQL
acceptance or execution.

Recheck CI at this head; the earlier unit-test, bundles/revision smoke and
aggregate failures are historical until refreshed. Complete the PR's
functional QA and obtain review approval.
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

D55 audit additions (2026-11-06). Each construct below remained in upstream
`4.1` at `148e09f015` and is masked in the PoC by a Stage 4 override,
generator rewrite or PoC-only commit (DIALECT.md section 4). The shared fix
must be valid SQLite and contain no PostgreSQL SQL. Base-DDL changes alter
the schema hash, so the 4.1 beta release policy applies.

| Finding | Candidate scope / next action |
|---|---|
| X29 predicate-free joins | Five schema views: `v_model_all`, `v_secret_permission`, `v_secret_owner`, `v_machine_target_agent_version`, `v_unit_target_agent_version`. Use explicit `CROSS JOIN` (precedent X38, #23695). Coordinate `v_model_all` with #23751. Delete the matching override hunks afterwards. |
| X16 `v_user_last_login` | Remove the bare column from `controller/sql/0019-model-last-login.sql` while keeping a `time.Time`-scannable result on dqlite. Verify the last-login read paths on SQLite and dqlite. |
| Duplicate FK name | Rename one `fk_offer_connection_uuid` on `application_remote_consumer` (`0034-cross-model-relation.sql`). Name-only change; check that nothing refers to the name. |
| X31 constraint order | `v_application_constraint` orders by implicit `rowid`, and `GetApplicationConstraints` documents insertion order, so treat order as a contract. Add an explicit sequence column to `constraint_tag`, `constraint_space` and `constraint_zone`, written from the input slice index. Every writer (`application/state/application.go`, `machine/state/placement.go`, `model/state/model/state.go`) deletes and reinserts the whole list, so the index is enough; `network/state/space_remove.go` deletes single rows, which only leaves gaps. Order the view by the new column. Check that export and import preserve the order. Removes D17's PostgreSQL-only identity columns. |
| X7 remaining functions | `IFNULL` (4 files), `GROUP_CONCAT`, `INSERT OR IGNORE`, `DATETIME('now')` in removal and upgrade state. Port the non-lease parts of PoC `8930466847` per package. Separate from #23751 (`IIF`). |
| D16 lease time | Decided: keep database time. `lease/state` claim/extend use `datetime('now', $duration)` and `leaseexpiry` compares `expiry < datetime('now')`. Port `CURRENT_TIMESTAMP` for the comparison and compute the expiry from the database time read in the transaction (PoC 05-lease-db-time), keeping whole-second text. Do not port the PoC's earlier controller-clock binding. A clock that never runs faster than real time across leader changes, clock steps and outages (like the old Juju global clock) is a separate design for both backends. |
| Lease migration `date('now')` | `domain/lease/state/migration.go` selects leadership leases with `expiry >= date('now')`: midnight UTC today, so it also exports leases that expired earlier today. PostgreSQL accepts it only as a function-style cast. Confirm intent (likely `expiry > CURRENT_TIMESTAMP`, or all leases) and add a test with an expired lease. |
| X8 bare `user` | Quote `user` in seven state files and 15 schema DDL references. Valid SQLite; removes the generator's quoting. |
| X5 scalar `MAX` | `SetLastLogTransferTime` (`modelmigration/state/model/state.go`); use `CASE` as in PoC `1e25d5bf39`. |
| X2 empty `IN` | sqlair renders an empty slice as `IN ()`. Choose portable rendering in sqlair or caller guards; inventory callers that can pass empty slices. Removes the Stage 3 driver rewrite. |
| `BOOLEAN DEFAULT 0` | `flag.value` (`controller/sql/0016-flag.sql`); use `FALSE`. Audit seed rows that write `0` / `1` to `BOOLEAN` columns. |
| `STRFTIME` defaults (tentative, D56) | Seven `STRFTIME('%Y-%m-%d %H:%M:%f', 'NOW', 'utc')` defaults in `0003-changelog.sql`, `0001-changelog.sql`, `0015-charm.sql`, `0018-machine.sql` and `0028-cleanup.sql`. Possible change to `CURRENT_TIMESTAMP`, which is portable, always UTC, and matches the whole-second text that `DATETIME('now', 'utc')` already writes. Not committed: it drops millisecond precision on SQLite. No reader needs that precision today, but keeping it costs little (DIALECT.md section 4). Decide before proposing. |
| Charm `create_time` order | `GetLatestPendingCharmhubCharm` (`application/state/charm.go`) takes the first row of `ORDER BY c.create_time DESC`, which depends on the default because `setCharmState` omits `create_time`. Charms created at the same instant order arbitrarily, and import binds Go-formatted times into the same text column. Bind the time from the domain clock in UTC and add a deterministic tiebreak, or order by an explicit key. Independent of the `STRFTIME` decision. |
| X25 `allocate_public_ip` | Add to the #883 removal checklist above: when Juju bundles the dqlite fix, make the column `BOOLEAN` and its export type `*bool`, removing the Stage 4 override. |

### X16 dbrepl output follow-up

[#23757](https://github.com/juju/juju/pull/23757) merged on 2026-11-09.
Its shared renderer preserves both queries and REPL output; implementation,
review and revision-specific validation are in
[UPSTREAM-DONE.md](UPSTREAM-DONE.md#dbrepl-output-refactor).
Live REPL QA was not recorded. Dbrepl is outside the PostgreSQL PoC scope.

Not stock-upstream tasks: the PostgreSQL `constraint.allocate_public_ip`
export-type mismatch needs backend-aware generation until the X25 candidate
above lands; the intentional
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
