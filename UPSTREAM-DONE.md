# Upstream work completed and rationale

This records implemented upstream work, decisions and validation, including
changes still in review. **Implemented is not landed or reintegrated.**
Statuses below are the recorded 2026-10-30 snapshot, not a fresh GitHub check.

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

The equality change to the storage lifecycle triggers changes the schema
patch hash: model databases created by an earlier 4.1 beta fail that hash
check. Released 4.0.15 DDL is unchanged. Do not describe this as preserving
all earlier-beta database compatibility.

## Implemented upstream, still in review

| Branch / PR | Finding | Implemented scope and rationale |
|---|---|---|
| `4.1-model-config-precedence`, #23683 | U7 | Exclude agent stream/version from competing config rows so authoritative `agent_version` values win regardless of UNION sorting or row order. |
| `4.1-operation-query-portability`, #23693 | X34/X33 | Separate boolean-comparison and text-size-expression commits, preserving filtering and pruning thresholds. Text size is not a backend-independent stored-bytes measure. |
| `4.1-crossmodel-secret-revisions`, #23688 | U11 | Any remote unit behind makes a secret reportable; return its latest revision once and join through indexed offer connections rather than application-name prefixes. |
| `4.1-local-secret-revisions`, #23720 | U11 follow-up | Independently select latest local consumed-secret revisions in two query methods; this is not the five-query latest-revision task. |
| `4.1-nullable-primary-keys`, #23710 | U15, charm | Require mount container keys and key manifest bases by manifest position, retaining nullable attributes. |
| `4.1-application-exposure-scope`, #23723 | U15, exposure | Separate application-wide wildcard exposure from endpoint-specific rows and require endpoint IDs in endpoint tables. |
| `4.1-lookup-key-not-null`, #23724 | U17 | Explicitly require INT lookup keys and reject absent payload IDs rather than silently treating them as valid zero IDs. |

Remaining review actions belong in the plan, not this history. GitHub is the
source of truth for live review, CI and merge status.

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

Published refreshed revisions: base `8369e6e2a9`, charm `ba21f48ad3`
(`7001a1b74b` plus diagnostic/generator follow-up), exposure `f31de64e52`
(`4d9518117c` plus deletion diagnostic). Earlier split heads were charm
`4df04695d8` and exposure `3c90a7ded3` on `9ea82c03bd`, preserved in
`backup/charm-before-base-refresh-20261030` and
`backup/exposure-before-base-refresh-20261030`. Original combined work is in
`backup/nullable-primary-keys-before-split-20261030`.

After explicit approval, both heads and exact descriptions were published:
[charm description](PR-4.1-nullable-primary-keys.md),
[exposure description](PR-4.1-application-exposure-scope.md), and
[context-only reply](COMMENT-4.1-nullable-primary-keys-review.md).
No review threads were resolved. Exposure targets upstream `4.1` because
GitHub cannot target the fork-only charm base; its description discloses
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

Refreshed heads: nine charm and 13 exposure generator/schema/import/recovery
and affected consumer race packages passed, as did focused exposure-state.
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

Both patches were cherry-picked into `pg-poc/05-queries` on 2026-10-26 with
matching patch IDs. That does not mean their eventual upstream merges are
reconciled. The register records PostgreSQL results and the separate
remaining secret-state fixture failures.

## Lookup-key NOT NULL decisions

PR #23724 (`3157cdb288`) is independent on refreshed `8369e6e2a9`, not
stacked on U15. At that base, SQLite CREATE TABLE/`pragma_table_info`
inventory found 48 model and 17 controller single-column `id INT PRIMARY
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
Recovery already validates presence/NULL against scalar fields; real
model/controller decode/replay still needs evidence after generation.

The review requests expanded nullable payload validation and a generic
helper for typed lookup deltas. Implementation is not proof those requests
are complete. The register holds the table inventory; remaining acceptance
checks and scoped/global guard integration are in the plan. No PostgreSQL
experiment was performed for U17 in the recorded session.

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
