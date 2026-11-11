# 4.1-sequence-upsert-target

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Head `696c116f79` replaces RETURNING with a checked upsert and same-transaction lookup to preserve retryable errors. Current reported checks have no failures or pending jobs; Simon's approval predates this head. Complete functional QA and review. Preserve zero-based allocation, contiguous batches, namespace isolation, rollback and import continuity.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-PLAN.md`:

[4.1-sequence-upsert-target](https://github.com/jameinel/juju/tree/4.1-sequence-upsert-target), [juju/juju#23731](https://github.com/juju/juju/pull/23731) | [696c116f79](https://github.com/jameinel/juju/commit/696c116f7943d16e6d1706fc857c561f90ee8aa3): checked upsert plus same-transaction lookup and stale-snapshot regression, superseding `76b4a04f09`'s RETURNING implementation | Review and functional QA remain; check CI again before acting.

From `UPSTREAM-DONE.md`:

`4.1-sequence-upsert-target`, [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | Target namespace conflicts explicitly, qualify the stored value and share a scalar allocator. At `696c116f79`, a checked upsert and same-transaction lookup replace `76b4a04f09`'s RETURNING implementation to preserve retryable errors. Preserve zero-based, contiguous and namespace-isolated allocation, rollback and import continuity.

### Upstream register snapshot

`4.1-sequence-upsert-target` | X3/X4 | [#23731](https://github.com/juju/juju/pull/23731) | `696c116f79` | open; current checks have no reported failures or pending jobs; earlier human approval predates head; functional QA remains

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

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
[UPSTREAM-DONE.md](#sequence-allocation-and-review-follow-ups).
Other X3/X4 operations remain available for independent scoping.

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Sequence allocation and review follow-ups

The independent implementation started at upstream
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48`. Initial commit `cfa8676090`
targets namespace conflicts and qualifies the stored value; follow-up
`74aabf93b4` collects concurrent-test results only after the entire
transaction succeeds. Earlier published head `76b4a04f09` shares a private
scalar allocator between `NextValue` and `NextNValues`, using RETURNING instead
of a second SELECT. A zero-count batch remains a no-op; allocation retains
namespace isolation, zero-based contiguous ranges, rollback and failure
detection when writes produce no row.

On 2026-11-06, published head `696c116f79` replaced RETURNING with a
checked, targeted upsert followed by a same-transaction lookup. The code
documents separate dqlite response and Sqlair row-iteration error defects;
the workaround must be reviewed when both are fixed. It adds a
stale-snapshot retryability regression for scalar and batch allocation,
retaining suppressed-write detection. The D55 review inspected this change
and verified the GitHub head; it did not rerun tests or CI. The earlier
RETURNING-head checks below are not validation of this revision.

The earlier implementation record reports the following checks using
shared `_deps`, stock SQLite/Dqlite fixtures and `dqlite,libsqlite3` tags.
These are historical results, not fresh validation of `76b4a04f09`:

- Before and after the original SQL change, `make run-go-tests
  TEST_PACKAGES=./domain/sequence/state
  TEST_FILTER='"TestSequenceSuite|TestStateSuite/TestImportedSequencesContinue"'
  TEST_ARGS='-p 2 -race'` passed; `TEST_PACKAGES='./domain/sequence/...'`
  with the same test args also passed.
- `make pre-check` passed but skipped static analysis.
  `RUN_SUBTEST=run_go make static-analysis
  STATIC_ANALYSIS_JOB=test_static_analysis_go`, formatting and diff checks
  passed.
- State test-binary compilation used `make run-go-tests
  TEST_PACKAGES=./domain/sequence/state
  TEST_ARGS='-p 2 -race -c -o <session-binary>'`.
  Bounded `timeout 31 stress -p 2 -timeout 30s <session-binary>` runs
  reported 10 original allocation/import iterations and 36 later
  `TestSequenceSuite/TestSequenceConcurrentMixedAllocation` iterations,
  zero failures before intentional outer timeouts (two iterations active
  at each final report). Binaries were removed.
- A later session record also reports a passing full state race suite and
  compilation. Do not treat those reports as evidence that the current
  failing CI jobs or functional QA are resolved.

The fixtures cover mixed/concurrent allocation, namespace variants,
rollback, suppressed writes and imported-sequence continuation through
export. They do not establish PostgreSQL SQL acceptance or execution.
PostgreSQL execution, full repository tests and live bootstrap/migration QA
were not part of those recorded checks. Current review, CI triage and
functional QA belong in this record's current-state section.
