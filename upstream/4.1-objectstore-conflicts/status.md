# 4.1-objectstore-conflicts

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-30; merge `75ddfcbaf0`

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [reply-review.md](reply-review.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-objectstore-conflicts`, #23694; `4.1-agentbinary-conflict`, #23728 | Aligned: SQL resolves intended uniqueness conflicts without continuing after a statement error. UUID/content conflicts and immutable registrations retain their domain meanings. Shared helper/refactoring and error mapping do not substitute for the SQL fix.

From `UPSTREAM-DONE.md`:

`4.1-objectstore-conflicts`, #23694 | X36 | Use one conflict clause and distinguish UUID reuse from content already stored; details below.

### Upstream register snapshot

`4.1-objectstore-conflicts` | X36 | [#23694](https://github.com/juju/juju/pull/23694) | `53feca2259` | merged 2026-10-30; merge `75ddfcbaf0`

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

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

The commit, [description](../../archive/pr-descriptions/PR-4.1-objectstore-conflicts.md) and
[reply](reply-review.md) were reviewed locally and
published only after explicit approval. `gh pr edit` failed on deprecated
classic-project GraphQL fields; the approved body was updated with
`gh api --method PATCH repos/juju/juju/pulls/23694`. No additional threads
were changed. Further publication needs fresh approval. PoC reconciliation
on 2026-11-05 adopted upstream's UUID-reuse classification and dropped the
duplicate patch, with default-backend validation only. The upstream merge
alone did not establish that reconciliation or PostgreSQL acceptance.
