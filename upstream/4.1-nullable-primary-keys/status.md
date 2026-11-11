# 4.1-nullable-primary-keys

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Simon approved; await the remaining GitHub review gate and land charm before exposure. Required container key, manifest-position key and duplicate-position diagnostics/tests are implemented.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)
- [reply-review.md](reply-review.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-nullable-primary-keys`, #23710 | U15, charm | Require mount container keys and key manifest bases by manifest position, retaining nullable attributes.

### Upstream register snapshot

`4.1-nullable-primary-keys` | U15 (charm keys) | [#23710](https://github.com/juju/juju/pull/23710) | `838abf41ea` | open; current macOS and aggregate checks passed; human approval exists, GitHub review required; land before exposure; cherry-picked onto PoC Stage 5 (D43)

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

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
Later review follow-ups and rebases changed the branch tips; the branch records
own the current heads.

After explicit approval, both heads and exact descriptions were published:
[charm description](pr.md),
[exposure description](../4.1-application-exposure-scope/pr.md), and
[context-only reply](reply-review.md).
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
[base refresh](#nullable-primary-key-base-refresh-2026-10-30)
and [initial split](#nullable-primary-key-split-validation-2026-10-30-pre-refresh).

<!-- Migrated from UPSTREAM-FIXES.md; historical evidence retains its revision. -->

### Nullable primary-key base refresh (2026-10-30)

Refreshed upstream and the local `4.1` base to `8369e6e2a9` (merged
integer-schema PR #23680). Rebased charm commits to `7001a1b74b` and
`ba21f48ad3`; restacked exposure to `4d9518117c` and `f31de64e52`.
Both branches retain only their assigned concern. Original split heads are
preserved in `backup/charm-before-base-refresh-20261030` and
`backup/exposure-before-base-refresh-20261030`. Upstream integer conversions,
full-row tests and schema changes were retained, not replaced by older
generated files. The exposure transform conflict was resolved by generation
from the combined schema; regeneration is idempotent on both branches.

Verification on the refreshed revisions:

- Base and final stack each passed
  `timeout 600s make go-build GOFLAGS=-p=2` with existing `_deps`, building
  all six linux/amd64 agent/client targets. Dependencies were not installed.
- Charm: `make run-go-tests` with
  `TEST_PACKAGES="./generate/export ./domain/schema ./domain/modelimport/... ./domain/recovery/..."`
  and `TEST_ARGS="-race -count=1 -p=2 -timeout=8m"` passed all nine packages.
  Focused charm import stress: 132 completed runs, zero failures before
  the intentional 31-second timeout.
- Final exposure: `make run-go-tests` with
  `TEST_PACKAGES="./generate/export ./domain/schema ./domain/application ./domain/application/service ./domain/modelimport/... ./domain/recovery/... ./domain/network/state ./domain/removal/state/model"`,
  `TEST_ARGS="-race -count=1 -p=2 -timeout=12m"` and `TEST_FILTER=Test`
  passed all 13 packages. The first redirected attempt had no package-result
  evidence; the explicit Test selector rerun above established coverage.
- Exposure state: `TEST_PACKAGES="./domain/application/state"`,
  `TEST_ARGS="-race -count=1 -p=2 -timeout=4m"`,
  `TEST_FILTER=TestExposedStateSuite` passed (10.5 seconds).
  Exposure watcher stress: 6 completed runs, zero failures before the
  intentional 31-second timeout. Temporary stress binaries were removed.
- Both `make pre-check` invocations passed but skipped static analysis;
  explicit `golangci-lint run --build-tags=dqlite,libsqlite3` over the affected
  package sets reported zero issues. Non-generated Go files were checked
  with gci and the worktrees are clean.
- Scoped diff whitespace checks pass. Full branch `git diff --check` reports
  two new generated `WHEN ` lines inheriting the existing trigger template's
  trailing space. They were left generator-consistent, not hand-edited or
  used to expand this PR into a trigger-formatting refactor.
- IDE build validation succeeds for the open base project, with limited
  diagnostics; the unopened exposure/charm worktrees rely on actual Go
  compilation/tests and the successful binary builds. Live-controller QA,
  PostgreSQL tests and the entire application-state suite were not rerun.

Publication completed after explicit approval of all content/actions:

- Atomically force-updated `jameinel/juju:4.1-nullable-primary-keys` from
  `43eed70758` to `ba21f48ad3` using an explicit lease; pushed the new
  `4.1-application-exposure-scope` branch at `f31de64e52`.
- Replaced the title/body of
  [juju/juju#23710](https://github.com/juju/juju/pull/23710) with the reviewed
  charm-only description.
- Created [juju/juju#23723](https://github.com/juju/juju/pull/23723), not draft,
  targeting `4.1` with the reviewed stacked exposure description.
- Posted the [contextual reply](https://github.com/juju/juju/pull/23710#issuecomment-6527752522),
  restricted by a sub-agent to the review's three suggestions and actual
  changes. It does not discuss the independent lookup-key plan.
- Read both PRs and the comment back and verified exact approved bodies,
  titles, branch heads, base `4.1` and non-draft status. New-head CI has not
  been assessed. No review threads were resolved or plan changes pushed.

Further published edits require fresh explicit approval.

<!-- Migrated from UPSTREAM-FIXES.md; historical evidence retains its revision. -->

### Nullable primary-key split validation (2026-10-30, pre-refresh)

Approved local split: charm-only `4df04695d8` for juju/juju#23710, exposure
`3c90a7ded3` stacked on it. This initial extraction retained base
`9ea82c03bd`; it was later refreshed and verified as recorded above.
These historical results do not establish coverage of the refreshed head.
No branches, PR content or plan changes have been pushed.

Charm validation:

```sh
make rebuild-export
make run-go-tests \
  TEST_PACKAGES="./generate/export ./domain/modelimport ./domain/modelimport/transformer/transforms/to_v4_1_0 ./domain/schema ./domain/recovery/state" \
  TEST_ARGS="-race -count=1 -p 2 -timeout 8m" TEST_FILTER=Test
# Generator test initially failed to compile (wrong helper name); fixed.
make run-go-tests TEST_PACKAGES="./generate/export" \
  TEST_ARGS="-race -count=1 -p 2 -timeout 3m" TEST_FILTER=TestGeneratorSuite
# Pass; the other packages in the earlier command all passed.
make pre-check
# Pass, but STATIC_ANALYSIS is disabled by default.
golangci-lint run ./generate/export/... ./domain/modelimport/...
# Zero issues.
```

Exposure validation before restack: complete race suites passed for
`./domain/schema`, `./domain/application`, `./domain/application/service`,
`./domain/network/state`, `./domain/removal/state/model`,
`./domain/modelimport`, and `./domain/recovery/state` using
`make run-go-tests TEST_ARGS="-race -count=1 -p 2 -timeout 8m"`.

After resolving handwritten conflicts and regenerating from the combined
schema, these checks passed:

```sh
make rebuild-export
make run-go-tests \
  TEST_PACKAGES="./generate/export ./domain/modelimport/... ./domain/recovery/..." \
  TEST_ARGS="-race -count=1 -p 2 -timeout 8m" TEST_FILTER=Test
make run-go-tests TEST_PACKAGES="./domain/application/state" \
  TEST_ARGS="-race -count=1 -p 2 -timeout 3m" \
  TEST_FILTER=TestExposedStateSuite
make pre-check
golangci-lint run ./generate/export/... ./domain/application/... \
  ./domain/modelimport/... ./domain/network/state/... \
  ./domain/removal/state/model/... ./domain/schema/...
git --no-pager diff --check
```

The first combined post-restack command also selected the entire
application-state package. That package reached its cumulative 8-minute
timeout while `TestUnitStateSuite/TestGetUnitWorkloadVersion` was running;
its full suite is **not** verified green. The focused exposure-state suite
passed afterwards (27.7 seconds), including failure injection for all four
exposure tables and rollback assertions.

Race binaries compiled with `make run-go-tests -c` were stressed using
`timeout 31s stress -p 2 -timeout 30s`: charm import selector
`TestImportSuite/.*Charm` reported 12 completed runs, zero failures;
exposure watcher selector `TestWatcherSuite/TestWatchApplicationExposed`
reported 6 completed runs, zero failures. Exit 124 is the intentional
outer timeout. Session-local binaries were removed afterwards.

Formatting used `gci` on touched non-generated Go files. IDE build calls
could not target these unopened worktrees; successful Go race tests supply
compilation evidence instead. Existing `_deps` were reused after the
initial missing-musl attempt was stopped; no dependency manifest changed.
The final stacked exposure tree also passed
`timeout 600s make go-build GOFLAGS=-p=2` using the existing `_deps`:
`jujuc`, `containeragent`, `pebble`, `jujuagentd`, `juju` and
`juju-metadata` built successfully for linux/amd64. Live-controller QA
and PostgreSQL tests were not run.

## U15 provenance snapshot (2026-11-11 migration)

**Cherry-picked onto `pg-poc/05-queries` 2026-11-05 (D43), PRs still open:** charm `9995f41b5a`→`6ffa7fa732`, `838abf41ea`→`7020bea305`; exposure `4d9518117c`→`7bd38bb203`, `f31de64e52`→`a43e38d25f`, `4e66f7c449`→`1e0bf5dfd4`; DDL regenerated in `52b6ba047c`. `6ffa7fa732` and `7bd38bb203` differ in patch ID only through the Stage 2 `domain/schema/dqlite/` path move (the `domain/schema/model.go` triggergen line gained the two new tables by hand); the others match. #23723's branch also carries #23710's older charm commits, which were not picked. PostgreSQL bootstrap not yet re-run. Earlier history: the initial split was published on 2026-10-30 from upstream `8369e6e2a9`. Current heads are charm `838abf41ea` in [#23710](https://github.com/juju/juju/pull/23710) and exposure `4e66f7c449` in [#23723](https://github.com/juju/juju/pull/23723); both remain open, and exposure must be rebased/regenerated after charm lands. Exposure on all endpoints moves to application-wide `application_exposed_space` / `_cidr` tables, mirroring bindings, and endpoint tables require the endpoint; the view still reports wildcard rows with a NULL endpoint. Mount container keys become `NOT NULL`; manifest bases use `(charm_uuid, array_index, nested_array_index)` and reject duplicate legacy positions with an identifying diagnostic. Exposure owns the combined schema guard. Legacy deltas split exposure scope and deduplicate wildcard rows; space removal also avoids alpha-duplicate UNIQUE failures.
