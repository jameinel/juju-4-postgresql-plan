# Upstream work completed and rationale

This records implemented upstream work, decisions and validation, including
changes still in review. **Implemented is not landed or reintegrated.**
The landed/in-review inventories are synchronized with GitHub on
2026-11-04. Validation and publication narratives retain their recorded
dates and revisions; they are not new checks of the latest heads.

- [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md): remaining work and acceptance checks.
- [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md): finding register, source provenance,
  current upstream disposition and eventual PoC reconciliation evidence.
- [UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md): reusable reintegration
  procedure. Landing upstream does not establish PoC reconciliation.

Keep new execution history and decision rationale here; keep pending actions
in the plan and finding/status mappings in the register. Historical heads
and test runs below do not establish coverage of later revisions.

## Landed upstream fixes

| Branch / PR | Finding | Work and rationale |
|---|---|---|
| `4.1-sqlite-integers`, #23680 | U1/U2/U12 | Correct application unit counts and integer lookup/relation-status declarations. SQLite coercion hid mismatched TEXT IDs; legacy string IDs are converted and exports use integer types. |
| `4.1-binary-key-columns`, #23682 | U8 | Store activation, bakery and macaroon root keys as BLOBs, with matching export/recovery and legacy compatibility. Raw random bytes are not text. |
| `4.1-keymanager-insert-id`, #23684 | U13 | Obtain the key ID from the inserted row. A zero-row insert for an unknown algorithm must not reuse a prior `LastInsertId` and associate an unrelated key. |
| `4.1-controller-node-id-text`, #23685 | U3/U5 | Bind decimal text to the TEXT node-ID column, including high-bit and max-uint64 values; correct changestream and upgrade fixtures. Do not narrow IDs to signed INT. |
| `4.1-lease-migration-tests`, #23686 | U4 | Assert seed success and use bound timestamps so held/expired lease tests exercise actual rows rather than silently passing on empty setup. |
| `4.1-removal-force-boolean`, #23690 | U9 | Match boolean writers/readers with BOOLEAN storage and bool exports; convert legacy 0/1 and reject other integers. |
| `4.1-changestream-latest-row`, #23691 | X16 | Explicitly select the latest change ID and join to its row, avoiding ungrouped columns beside MAX. This does not fix the separate dbrepl query. |
| `4.1-dbrepl-latest-row`, [#23730](https://github.com/juju/juju/pull/23730) | X16 | Apply explicit latest-ID selection and row join-back to the separate dbrepl change-stream query, preserving grouping, bounds, ordering and displayed fields. Merged 2026-11-04. |
| `4.1-local-secret-revisions`, [#23720](https://github.com/juju/juju/pull/23720) | U11 follow-up | Independently select latest local consumed-secret revisions in two query methods, with empty/multiple-secret/query-round-trip regressions and redundant deduplication removed. Merged 2026-11-04; the five-query latest-revision task remains separate. |
| `4.1-dbaccessor-pool-cleanup`, [#23726](https://github.com/juju/juju/pull/23726) | U14 | Close unretained pools on setup and reopen failures while preserving caller/worker ownership of retained pools. Merged 2026-11-04. |
| `4.1-blockcommand-order`, #23692 | U10/X31 | Replace implicit rowid order with explicit block-type order. This is an agreed observable ordering change, not a syntax-only rewrite. |
| `4.1-objectstore-conflicts`, #23694 | X36 | Use one conflict clause and distinguish UUID reuse from content already stored; details below. |
| `4.1-sql-standard-syntax`, #23695 | X37/X38 | Standard equality and an explicit CROSS JOIN to the one-row fulfilment aggregate. Cover unused/shared-key deletion and zero-fulfilment detach info. A third commit changes storage lifecycle trigger equality. |
| `4.1-sql-test-literals`, #23696 | U6 | Replace double-quoted string literals in fixtures and production SQL with standard single-quoted strings, preserving meaning. |

PR #23680 merged on 2026-10-30 at head `0e9be8ddd7`. Its review follow-up
added full-row conversion coverage, invalid-ID classification, explicit
view row counts and schema-evolution documentation. Race tests, 52 stress
iterations and targeted lint passed; live QA was not run. Details:
[integer review evidence](COMMENT-4.1-sqlite-integers-review.md).

PR #23691 merged on 2026-10-30 as `ece31c289c6a`; PR #23694 merged that day
as `75ddfcbaf0b3`. Other landed heads and historical source mappings remain
in [the register](UPSTREAM-FIXES.md#upstream-review-status).

PR [#23730](https://github.com/juju/juju/pull/23730) merged on 2026-11-04
as `f7b596bf5cb9b51aa44ec000d09b9f495aac29ad`, at head `49a1bdf808`.
The latest-row SQL task is complete. Extracting the duplicated dbrepl row
scanning and tabwriter output remains a separate candidate in
UPSTREAM-PLAN.md. No PoC integration was performed; dbrepl remains outside
the PostgreSQL PoC scope.

PR [#23720](https://github.com/juju/juju/pull/23720) merged on 2026-11-04
as `edb2aef5b9f1e4cf7c5ceb890fd137d922048c30`, at head `ee07d7ccb5`.
PR [#23726](https://github.com/juju/juju/pull/23726) merged that day
as `5259edff4329ca4a1de80d77e0d614b832d4ad72`, at head `ee55bab5f4`.
Both tasks are complete upstream. Their landed patches still need
reconciliation against the earlier PoC implementations; no integration or
equivalence verification was performed as part of this tracking update.

The equality change to the storage lifecycle triggers changes the schema
patch hash: model databases created by an earlier 4.1 beta fail that hash
check. Released 4.0.15 DDL is unchanged. Do not describe this as preserving
all earlier-beta database compatibility.

## Implemented upstream, still in review

| Branch / PR | Finding | Implemented scope and rationale |
|---|---|---|
| `4.1-model-config-precedence`, #23683 | U7 | Exclude agent stream/version from competing config rows so authoritative `agent_version` values win regardless of UNION sorting or row order. |
| `4.1-operation-query-portability`, #23693 | X34/X33 | Portable boolean comparisons and text-size expressions, plus the reviewed fix that treats each NULL column as zero instead of discarding the entire row from the estimate. Filtering is preserved; size/pruning decisions for nullable rows intentionally change. Text size is not stored bytes. |
| `4.1-crossmodel-secret-revisions`, #23688 | U11 | Any remote unit behind makes a secret reportable; return its latest revision once through indexed offer connections. Tracking uniqueness and upserts are scoped by `(offer_connection_uuid, secret_id, unit_name)`; synthetic-unit cleanup uses one CTE-backed DELETE. |
| `4.1-nullable-primary-keys`, #23710 | U15, charm | Require mount container keys and key manifest bases by manifest position, retaining nullable attributes. |
| `4.1-application-exposure-scope`, #23723 | U15, exposure | Separate application-wide wildcard exposure from endpoint-specific rows and require endpoint IDs in endpoint tables. |
| `4.1-lookup-key-not-null`, #23724 | U17 | Explicitly require INT lookup keys and reject absent payload IDs rather than silently treating them as valid zero IDs. |
| `4.1-agentbinary-conflict`, [#23728](https://github.com/juju/juju/pull/23728) | U16/X18 | Use `ON CONFLICT (version, architecture_id) DO NOTHING` and compare the retained registration in both databases, preserving idempotency, immutability and transaction usability. |
| `4.1-lease-pin-conflict`, [#23729](https://github.com/juju/juju/pull/23729) | X18 | Resolve competing pins with the explicit lease/entity conflict target. Preserve repeated-pin and missing-lease behaviour; reset the test helper's captured results on transaction retries. This is portability, not a current dqlite defect. |
| `4.1-sequence-upsert-target`, [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | Target namespace conflicts explicitly and qualify the sequence value in single/batch allocation upserts. Preserve zero-based, contiguous and namespace-isolated allocation, rollback and import continuity. |

Remaining review actions belong in the plan, not this history. GitHub is the
source of truth for live review, CI and merge status.

## Dbrepl latest-row selection (X16)

Implemented locally on `4.1-dbrepl-latest-row`, worktree
`../4.1-dbrepl-latest-row`, independently based on upstream 4.1
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48`. Commit
`49a1bdf8085abbde324bedce53bbc189ed2efcfb` changes only `worker.go` and
`worker_test.go` in `internal/worker/dbrepl`. After explicit approval, the
branch was pushed and published as
[juju/juju#23730](https://github.com/juju/juju/pull/23730), from
`jameinel:4.1-dbrepl-latest-row` to `juju/juju:4.1`. The
[local PR text](PR-4.1-dbrepl-latest-row.md) matches the published title/body.
The PR subsequently merged on 2026-11-04 as `f7b596bf5cb9`; the detailed
validation below records the original implementation, not a new test run.
No PoC integration was performed; dbrepl remains outside the PoC scope.

The assigned plan incorrectly described replication payload/progress.
Inspection showed that `changeStream` serves the interactive
`.change-stream` REPL command and prints rows; it has no replication state.
The user approved correcting that scope on 2026-10-30. Acceptance covers
the displayed header/fields and query meaning, not invented replication
behaviour. The separate changestream worker fix landed in juju/juju#23691;
this REPL query was not changed by it.

`MAX(c.id)` itself is standard SQL. The SQLite-specific behaviour was
selecting ungrouped edit type, namespace, key and timestamp from the row
supplying the maximum ID. The rewrite groups `change_log` by
`(namespace_id, changed)` to select maximum IDs, joins back to those rows,
then joins edit-type and namespace lookups. It preserves `id > 0`, latest
ID rather than latest timestamp, ascending ID ordering, all displayed
fields, coalescing and output formatting. Valid lookup references remain
enforced by the existing foreign keys; no schema/dependency change occurs.

Four regressions inspect the actual REPL output, normalising table
whitespace only. They cover multiple keys/namespaces, repeated keys whose
edit types/timestamps differ, timestamps reversed relative to IDs, empty
output, the first positive ID and excluded zero/negative-only keys. All
four pass with the original and rewritten SQL on both SQLite and actual
Dqlite. The final test helper was rechecked against the original SQL, then
the committed rewrite was restored; this is a portability improvement,
not a changed SQLite result contract.

### Validation on the local head

Commands ran from the new worktree, using its shared `_deps` symlink:

```sh
gofmt -w internal/worker/dbrepl/{worker.go,worker_test.go}
gci write --section standard --section default \
  --section 'Prefix(github.com/juju/juju)' \
  internal/worker/dbrepl/{worker.go,worker_test.go}
make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl \
  TEST_ARGS='-race -p 2 -count=1' \
  TEST_FILTER='TestDbReplSuite/TestChangeStream'
go test -race -p 2 ./internal/worker/dbrepl \
  -run 'TestDbReplSuite/TestChangeStream' -count=1
make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl \
  TEST_ARGS='-race -p 2 -count=1'
go test -race -p 2 ./internal/worker/dbrepl -count=1
make pre-check
golangci-lint run --concurrency 2 ./internal/worker/dbrepl/...
go build -p 2 ./internal/worker/dbrepl
git --no-pager diff --check
```

All passed. The Makefile supplies `dqlite,libsqlite3`, musl, CGO and linker
settings automatically. Final full package runs passed in 6.610s (Dqlite)
and 2.943s (SQLite). Initial targeted lint requested `strings.SplitSeq`
instead of `strings.Split` in the output helper; this was corrected and
lint rerun with zero issues. `make pre-check` skipped full static analysis.
The worktree was not an open IDE project; Go tests/build supplied build
validation.

Final actual-Dqlite stress used:

```sh
make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl \
  TEST_ARGS="-race -p 2 -c -o $binary"
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run TestDbReplSuite -test.timeout 30s
```

It completed 20 runs with zero failures before the intentional outer
exit 124. An earlier pre-lint-adjustment run completed 18 runs with zero
failures. The session-local test binary was removed. Full repository static
analysis/tests, full agent build, live interactive QA and PostgreSQL tests
were not run. Publication verification confirmed juju/juju#23730 open,
not a draft, targeting `4.1` at the recorded head; review/CI results have
not yet been assessed.

SQLite `EXPLAIN QUERY PLAN` on the existing changelog schema showed that
both versions scan the same qualifying source rows and use temporary
B-trees for grouping and ordering. The rewrite adds indexed fetches of
latest changelog rows, but performs edit-type/namespace joins only after
coalescing rather than for every source row. This is planner evidence,
not a benchmark or a guarantee of faster execution; when most keys have
only one row, the added fetches can add overhead. The command is an
interactive REPL path, not the production watcher.

## Dbaccessor pool cleanup (U14)

Implemented independently from refreshed upstream 4.1 base
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` in worktree
`../4.1-dbaccessor-pool-cleanup`, sharing `../juju/_deps`. Commit
`ee55bab5f471b1f0e19344a0dcb94a5a181808b3` changes only `tracker.go` and
`tracker_test.go` in `internal/worker/dbaccessor`. After explicit approval,
the branch was pushed to `jameinel/juju` and published as
[juju/juju#23726](https://github.com/juju/juju/pull/23726), targeting `4.1`.
The [local PR text](PR-4.1-dbaccessor-pool-cleanup.md) matches the published
title/body. No published comments or review threads were modified.

The tracker leaked pools after foreign-key pragma setup failure, model
initialisation failure and failed verification of replacement pools.
Cleanup closes failed startup pools and each failed replacement before
another reopen or return. The caller still owns the original retained
pool; successful replacements remain open to be adopted by the loop.
Operation errors are preserved and close errors are logged. Pool limits
remain three idle/open connections, unchanged from the 4.1 base; the older
limit of two visible in `main` is not part of this diff.

Nine regressions cover pragma failure, model metadata and migration
failures, initialised-model usability, intermediate/terminal replacement
failures, reopen failure, cancellation, caller-owned pools and successful
replacement transfer. Before production edits, six regressions failed
because leaked pools were still usable; the successful model case passed.
The caller-ownership and cancellation cases were added subsequently.

### Validation on the published head

Commands ran from the new worktree. Formatting used `gofmt` and:

```sh
gci write --section standard --section default \
  --section 'Prefix(github.com/juju/juju)' \
  internal/worker/dbaccessor/{tracker.go,tracker_test.go}
go test -race -p 2 ./internal/worker/dbaccessor -count=1
make run-go-tests TEST_PACKAGES=./internal/worker/dbaccessor \
  TEST_ARGS='-race -p 2 -count=1'
make pre-check
golangci-lint run --concurrency 2 ./internal/worker/dbaccessor/...
go build -p 2 ./internal/worker/dbaccessor
git --no-pager diff --check
```

All passed. `make run-go-tests` automatically enables `dqlite,libsqlite3`
and supplies the existing musl/CGO/library/linker environment; its full
package race run passed in 18.698s. `make pre-check` skipped full static
analysis; explicit targeted golangci-lint reported zero issues. The worktree
was not an open IDE project, so command-line compilation supplied build
validation. `make go-test-alias` is also available; no `go-test` target
exists.

SQLite-backed tracker stress used:

```sh
go test -race -p 2 -c -o "$binary" ./internal/worker/dbaccessor
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run TestTrackedDBWorkerSuite -test.timeout 20s
```

It completed 18 runs with zero failures. Actual Dqlite stress used the
existing static archives with host gcc:

```sh
export CGO_ENABLED=1
export CGO_CFLAGS="-I$PWD/_deps/dqlite-deps-4.0-amd64/include"
export CGO_LDFLAGS="-L$PWD/_deps/dqlite-deps-4.0-amd64 -Wl,-Bstatic -ldqlite -luv -llz4 -lsqlite3 -Wl,-Bdynamic -lm"
go test -race -p 2 -tags=dqlite,libsqlite3 -c -o "$binary" \
  ./internal/worker/dbaccessor
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run TestTrackedDBWorkerSuite -test.timeout 30s
```

It completed 10 runs with zero failures. Both stress commands ended with
intentional outer timeout exit 124; both session-local binaries were
removed. An initial host-gcc Dqlite attempt without static archive selection
and `-lm` failed linking; the corrected flags passed without installations.
The Makefile target avoids manually supplying this environment.

At the 2026-10-30 publication follow-up, GitHub reported the PR open and
mergeable, with review required and CI still in progress. No failed checks
were observed; completed checks included CLA, conventional commits, SQL,
arm64/macOS builds and migration. This is not a full-green CI result.
Repository-wide tests/static analysis, full agent build, external bootstrap
QA and PostgreSQL integration tests were not run for this upstream branch.

PoC Stage 8 already implements U14 as `b6a7690d80`. This independent
upstream implementation is not a cherry-pick or a PoC reconciliation.
PR juju/juju#23726 has now landed as `5259edff43`; compare its final
cleanup and tests with that commit while preserving later tracker/backend
adaptations, using the integration procedure. This reconciliation remains
pending; provenance and reconciliation status belong in the register.

## Agent-binary and lease-pin publication evidence

These are historical 2026-10-30 implementation checks, not new validation
of current PR heads. Both independent branches started at upstream
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` with shared `_deps`.

Session `2b06495a-18f9-4418-ba5c-b2e8fd908dd1` implemented
`4.1-agentbinary-conflict` in `../4.1-agentbinary-conflict`, restricted to
`domain/agentbinary/state/{controller,model}`. Implementation
`323807680f7a5249f65d2d7378c0c2158cb3edee` and comment-only follow-up
`6a66e5ed1be4cc0bde5fee269b6da1f87da56440` were pushed after approval
as juju/juju#23728. The revised description was approved and verified
against [the local text](PR-4.1-agentbinary-conflict.md).

Conflict-safe inserts and affected-row checks preserve error sentinels and
immutability. New tests cover same-transaction writes after both conflict
cases, original mapping retention and probe-write rollback. Validation:

- `make run-go-tests` with
  `TEST_PACKAGES='./domain/agentbinary/state/controller ./domain/agentbinary/state/model'`,
  `TEST_FILTER='Test.*StateSuite/TestAdd'`, `TEST_ARGS='-p 2 -race'`: passed.
- The same command without `TEST_FILTER`: both full state race suites passed.
- Each package compiled with `make run-go-tests`,
  `TEST_ARGS='-p 2 -race -c -o <session-binary>'`, then ran with
  `timeout 31 stress -p 2 -timeout 30s <session-binary> -test.run <suite/TestAdd>`.
  Controller completed 30 iterations and model 20, zero reported failures
  before intentional outer timeouts. Binaries were removed.
- `make pre-check` passed but skipped static analysis.
  `RUN_SUBTEST=run_go make static-analysis STATIC_ANALYSIS_JOB=test_static_analysis_go`
  passed, as did `gofmt`, prescribed `gci` and `git diff --check`.
  Other static-analysis jobs and full repository tests were not run.

Tests were not rerun for the comment-only follow-up; formatting/diff checks
passed. It describes immutable registration rather than naming a backend.
Validation used stock dqlite fixtures; no PoC reconciliation was performed.

Session `a646e393-debb-413a-851c-ca78d13c9a23` implemented
`4.1-lease-pin-conflict` in `../4.1-lease-pin-conflict`, restricted to
`domain/lease/state`. Initial published commit `373ab4f422` opened
juju/juju#23729 using [the local description](PR-4.1-lease-pin-conflict.md).
`make run-go-tests TEST_PACKAGES=./domain/lease/state/...` passed with
`-race -count=3` and `-race -count=20 -run TestStateSuite/TestPin`; lint
was clean. Regressions also passed with the old SQL because SQLite keeps
the transaction usable after the caught constraint failure. This is
portability, not a current dqlite defect; no production timing change
belongs in this branch. Later published follow-ups moved its head to
`c55d5839e5`; the historical checks above do not establish its validation.

## Nullable primary-key review split

SQLite permits NULL in non-integer primary keys and treats NULLs as distinct.
The exposure tables used NULL endpoint IDs for wildcard exposure, allowing
duplicates and causing the first PostgreSQL bootstrap to fail. Charm keys
had a separate schema-contract problem. The 2026-10-30 review requested two
concerns rather than one combined PR.

The existing `4.1-nullable-primary-keys` PR #23710 retained charm changes.
Exposure moved to `4.1-application-exposure-scope`, PR #23723, stacked on
charm. This is an explicit exception to independent-base branches: the
whole-schema primary-key guard passes only after both fixes and belongs to
exposure. No unrelated PoC code was included.

- Charm owns `0015-charm.sql`, schema regressions, mount export type,
  legacy mount/manifest-position deltas, constraint-only delta generator
  support, generated transforms/recovery and tests. Duplicate legacy
  manifest positions halt with the charm UUID and both indices, not a raw
  UNIQUE error; nullable manifest attributes remain valid.
- Exposure owns `0019-application.sql`, four-table writes/view, watcher
  namespaces/triggers, export/import/recovery, removal and space operations,
  tests and the combined primary-key guard. Application-wide space/CIDR
  tables mirror bindings; the view/API still expose the `""` wildcard with
  a NULL endpoint. Legacy import separates scopes and collapses duplicate
  wildcard rows. Reset-to-alpha avoids UNIQUE errors when alpha is already
  exposed. Deletion failures identify the SQL/table and roll back preceding
  deletions in the same transaction.
- Each branch was regenerated from its own schema. Exposure regeneration
  after restacking retained both concerns' deltas rather than choosing an
  entire generated file from one branch.
- U17 is independent: shared SQL/generated files are integration conflicts,
  not semantic prerequisites. Its INT-key-scoped guard can pass before U15;
  the global guard must eventually exempt only rowid-aliasing INTEGER keys.

2026-10-30 publication (historical revisions): base `8369e6e2a9`, charm
`ba21f48ad3` (`7001a1b74b` plus diagnostic/generator follow-up),
exposure `f31de64e52`
(`4d9518117c` plus deletion diagnostic). Earlier split heads were charm
`4df04695d8` and exposure `3c90a7ded3` on `9ea82c03bd`, preserved in
`backup/charm-before-base-refresh-20261030` and
`backup/exposure-before-base-refresh-20261030`. Original combined work is in
`backup/nullable-primary-keys-before-split-20261030`.
Later review follow-ups and rebases changed the branch tips; the register
owns the current heads.

After explicit approval, both heads and exact descriptions were published:
[charm description](PR-4.1-nullable-primary-keys.md),
[exposure description](PR-4.1-application-exposure-scope.md), and
[context-only reply](COMMENT-4.1-nullable-primary-keys-review.md).
No review threads were resolved at that publication. Exposure targets
upstream `4.1` because GitHub cannot target the fork-only charm base;
its description discloses
the stack and scoped comparison. Charm must land before exposure is rebased
and landed. Both fixes are required for PoC bootstrap reintegration.

### Validation and limits

Initial split: charm schema/import/recovery race suites passed after a
generator test-helper typo was fixed. Exposure schema/application/service/
network/removal/import/recovery race suites passed before restack; after
restack, generator/import/recovery and focused exposure-state suites passed.
The full application-state suite timed out cumulatively at eight minutes
in `TestUnitStateSuite/TestGetUnitWorkloadVersion`; it was not a full green
run. Charm-import stress reported 12 completed runs and exposure-watcher
stress 6, zero failures before intentional outer timeouts.

2026-10-30 refreshed heads: nine charm and 13 exposure generator/schema/
import/recovery and affected consumer race packages passed, as did focused
exposure-state.
Charm-import stress reported 132 completed runs and exposure-watcher stress
6, zero failures before intentional outer timeouts. Regeneration was
idempotent. Base and final stack passed `timeout 600s make go-build
GOFLAGS=-p=2` for all six linux/amd64 targets with existing `_deps`.

`make pre-check` passed but skipped static analysis; explicit focused,
build-tagged lint reported zero issues. Scoped handwritten whitespace
checks passed; generated trigger-template trailing whitespace was retained
rather than expanding the concern into template cleanup. Unopened worktrees
were validated by Go compilation/tests/builds, not an IDE build. Live QA,
PostgreSQL tests and the full application-state suite were not rerun.
Exact commands and publication verification:
[base refresh](UPSTREAM-FIXES.md#nullable-primary-key-base-refresh-2026-10-30)
and [initial split](UPSTREAM-FIXES.md#nullable-primary-key-split-validation-2026-10-30-pre-refresh).

## Secret-query decisions

The #23688 remote fix keeps the any-unit-behind query, indexed
`offer_connection_uuid` schema/export/import/recovery changes, consumer
cleanup and regressions together. Covering only a single consuming unit
would not establish the contract.

Migration warns and discards unresolvable remote revision-tracking rows,
including malformed names. These memoise consumed revisions; they are not
secret content, grants or core model structure. A normal read recreates
missing tracking at the latest revision; dropping tracking does not itself
emit a watcher event. Preserve valid tracking. This is a bounded exception
for reconstructible metadata, not permission to discard inconsistent model
data generally.

The extracted local portability fix owns only
`InitialWatchStatementForConsumedSecretsChange`,
`GetConsumedSecretURIsWithChanges` and
`TestConsumedSecretsChangesMultipleUnitsAndRevisions` in
`domain/secret/state`. It was split onto `4.1-local-secret-revisions` on
`9ea82c03bd`, commit `369fb64c2d`, and published as #23720. The regression
passes with original SQLite and portable SQL; full secret-state race tests
and focused lint passed. The remote review follow-up was `29fc497b6e`;
current recorded PR heads are in the register. Remote schema-dependent
fixtures stay in #23688. Its watcher still scopes by offered application;
per-connection watcher scoping is separate follow-up.

Earlier versions of both patches were cherry-picked into `pg-poc/05-queries`
on 2026-10-26 with matching patch IDs. That does not establish reconciliation
with #23720's landed head or the eventual #23688 merge. The register records
PostgreSQL results and the separate remaining secret-state fixture failures.

## Lookup-key NOT NULL decisions

The initial #23724 implementation (`3157cdb288`) was independent on
refreshed `8369e6e2a9`, not stacked on U15. At that base, SQLite
CREATE TABLE/`pragma_table_info` inventory found 48 model and 17 controller
single-column `id INT PRIMARY
KEY` declarations permitting NULL. INT is not the rowid-aliasing INTEGER
spelling, and duplicate NULL lookup keys were possible.

Explicit NOT NULL lets the existing export generator produce `int64`
instead of `*int64`; a blanket generator inference would hide the storage
defect. Only those keys and generated consumers change. Zero remains a
valid seed ID; absent/NULL IDs must not become zero during YAML decoding.
Consumed 4.0.12 model rows need typed deltas preserving every field and
validating before dereference; 4.0.12 types and historical DDL remain
unchanged. Controller exports start at 4.1.0, so no legacy controller delta
is required. Source changestream lookup rows remain excluded from import.
Recovery validates presence/NULL against scalar fields; model/controller
decode/replay and persistence regressions cover the regenerated types.

The published follow-up at `4e7341392f` adds a DDL-derived decoder-inventory
cross-check, direct `coreerrors.NotValid` classification, and the schema
guard's assertion outside its transaction. The proposed generic helper was
tried and reverted: each table still needed typed ID-accessor and mapping
closures, while `requiredLookupID` already centralizes NULL rejection.
Retain explicit deltas. The seven previously non-NULL keys are deliberate
inventory exceptions, not an expansion of this 65-key fix.
The [review response](https://github.com/juju/juju/pull/23724#issuecomment-6618410549)
records successful live 4.0-to-4.1 and 4.1-to-4.1 migrations. That does not
establish PoC integration. Remaining review/CI and combined-guard
reconciliation belong in the plan; the register owns current disposition.

## Objectstore review follow-up

Follow-up `53feca2259` on `4.1-objectstore-conflicts` (parent `87ac64185e`,
base `04bc64ef54`) landed with #23694. A reused UUID with different hashes
or size returns documented `objectstoreerrors.ErrMetadataUUIDAlreadyExists`,
preserving error text. It must remain distinct from
`ErrHashAndSizeAlreadyExists`, which callers interpret as already stored.
Both service put methods preserve the sentinel through wrapping. Hash and
size-only conflicts leave paths/hints unchanged. The table/projection-alias
review request was already implemented and its thread resolved.

Before the state-return fix, both existing different-hash UUID tests failed
`ErrorIs` against the bare error. Size-only UUID and both service-wrapper
regressions passed with the fix. Validation from the objectstore worktree:

```sh
make run-go-tests TEST_PACKAGES="./domain/objectstore/state" \
  TEST_ARGS="-race -count=1 -p 2" \
  TEST_FILTER="TestStateSuite/TestPutMetadata.*SameUUIDDifferentHashes"
make run-go-tests TEST_PACKAGES="./domain/objectstore/..." \
  TEST_ARGS="-race -count=1 -p 2"
make run-go-tests TEST_PACKAGES="./domain/objectstore/state" \
  TEST_ARGS="-race -p 2 -c -o $binary"
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run 'TestStateSuite/TestPutMetadata'
make pre-check
golangci-lint run ./domain/objectstore/...
git --no-pager diff --check
```

`$binary` was a removed session-local stress artifact. Stress reported 10
runs, zero failures before expected outer exit 124. Pre-check skipped static
analysis; explicit lint reported zero issues. The Makefile supplied
`-mod=readonly -tags=dqlite,libsqlite3`, CGO, existing musl/Dqlite and
external static linking with Go `go1.27.3`. gofmt and prescribed gci ran on
all four changed Go files. No dependencies were installed; deployment smoke,
PostgreSQL and IDE worktree builds were not run.

The commit, [description](PR-4.1-objectstore-conflicts.md) and
[reply](COMMENT-4.1-objectstore-conflicts-review.md) were reviewed locally and
published only after explicit approval. `gh pr edit` failed on deprecated
classic-project GraphQL fields; the approved body was updated with
`gh api --method PATCH repos/juju/juju/pulls/23694`. No additional threads
were changed. Further publication needs fresh approval. PoC reconciliation
must replace its UUID-reuse classification separately; it is not completed
by the upstream merge.
