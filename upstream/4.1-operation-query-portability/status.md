# 4.1-operation-query-portability

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

2026-11-20: The user reported completing the manual QA in [pr.md](pr.md)
and updated its commands. With user approval, the revised description was
published to [#23693](https://github.com/juju/juju/pull/23693) and verified
against the local draft. The user did not provide separate command output in
this session.

2026-11-17: The user authorized this session to take over the clean worktree
and approved an approximate static-query estimator in response to Simon.
The user subsequently requested constant row allowances and tests for
accuracy and schema drift. Branch `4.1-operation-query-portability` is at
commit `9b2e784d3c`, pushed with user approval to
`jameinel/juju:4.1-operation-query-portability` on 2026-11-17. The PR and
remote branch heads were verified at that commit after the push. Base
observed on GitHub is `5259edff43`; PR
[#23693](https://github.com/juju/juju/pull/23693) is open. Checks for the
new head were in progress or queued at 16:39 EST on 2026-11-17.

The follow-up removes the dynamic table sizing and count helpers, estimates
fixed-width fields per row and measures variable text. It preserves
referenced object-store accounting and the substance of the prior NULL fix.
Pruning decisions near the limit can change. With separate user approval,
the PR description was updated and
[Simon was replied to](https://github.com/juju/juju/pull/23693#discussion_r4606134121).
Both published texts were verified byte-for-byte against their local drafts.
The review thread was not resolved; await CI and Simon's review.

Fixed per-row constants are bound into the static SQL, with field counts
documented against the operation schema. A schema test flags added, removed
or retyped columns in the relevant tables. Per-row tests compare the fixed
allowances with representative textual values within 40%; a separate
end-to-end fixture compares the estimate to known values and object content
within 25% plus KiB rounding. Neither is a disk-size test.

Validation on the follow-up:

- `go test ./domain/operation/state -count=1`: passed (SQLite).
- `timeout 300s make run-go-tests TEST_PACKAGES=./domain/operation/state
  TEST_ARGS='-count=1 -p 2 -timeout 240s'`: passed (real dqlite, with
  `dqlite,libsqlite3` tags and bundled dependencies).
- `golangci-lint run -c .golangci.yml ./domain/operation/state/...`:
  passed; `txncheck "$PWD"`: passed. Both used `$HOME/go/bin` on `PATH`.
- `gci` and `gofmt` ran on all touched Go files; `git diff --check` passed.
  Full `make pre-check` and manual controller QA had not yet been run.

PostgreSQL diagnostic, 2026-11-17, on 16.15 (separate from supported-backend
acceptance): SQL from `9b2e784d3c` was executed with PostgreSQL positional
parameters in a transaction-scoped, PostgreSQL-typed subset of the operation
and object-store tables. The static table estimate and output-path query
passed with NULLs, Unicode and repeated references; all 32 boolean-filter
combinations passed with singleton filter values. The object-store cast
worked for zero, negative and >32-bit sizes. Direct `octet_length` calls on
integer, boolean and timestamp values were rejected, reproducing the
original type issue.

The branch's existing `WHERE uuid IN uuids` object-store query failed with a
PostgreSQL syntax error. Current `juju/juju:4.1` tip `1a0d0c78bc` contains
`02e9f73e49`, which replaces it with a subquery; that form passed the
diagnostic, and a read-only merge preview retained it alongside this PR's
cast. The PR API still reported its base SHA as `5259edff43`; the live
branch ref was checked separately. This was manually adapted SQL, not a
full PostgreSQL Juju/sqlair integration run. The temporary schema was rolled
back and its absence verified. The diagnostic SQL is retained in this
session's `files/postgresql-operation-check.sql`.

İsmail's error-context and NULL feedback was previously addressed; the
static query retains contextual errors and counts content despite nullable
fields. His signature and rebase requests were addressed before this review.
Van Cleef reported no issues or suggestions on the earlier head; its support
for the dynamic builder is superseded by Simon's maintainability concern.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)
- [reply-simon.md](reply-simon.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-operation-query-portability`, #23693 | X34/X33 | Portable boolean comparisons and text-size expressions, plus the reviewed fix that treats each NULL column as zero instead of discarding the entire row from the estimate. Filtering is preserved; size/pruning decisions for nullable rows intentionally change. Text size is not stored bytes.

### Upstream register snapshot

`4.1-operation-query-portability` | X34, X33 | [#23693](https://github.com/juju/juju/pull/23693) | `0fb2eb7194` | open; current macOS and aggregate checks passed; human approval exists, GitHub review required
