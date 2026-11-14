# sqlair-dollar-placeholders

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Claimed 2026-11-14 by session `020f1b3f-f7b5-4bd4-9ad2-e27661961d66`.
On 2026-11-14, migrated onto the published
`postgresql-test-infrastructure` tip `ccf6072c7527d0d897c26d4c7063c687cd0d92b6`
in worktree `canonical/sqlair/.worktrees/dollar-placeholders-on-postgresql`,
branch `dollar-placeholders-on-postgresql`, head `1213a73`. The layer has
`32953f1` (interpolation, cherry-picked from `9efd4ca`) and `1213a73`
(PostgreSQL coverage using the shared fixture). It is local and unpushed.
Target repository: [canonical/sqlair](https://github.com/canonical/sqlair).
The infrastructure branch is a dependency; it has been published to
`jameinel/sqlair:postgresql-test-infrastructure` at `ccf6072` but has no PR.
Native GitHub stacks do not support forked heads.

The original worktree `canonical/sqlair/.worktrees/dollar-placeholders`
and branch `dollar-placeholders` remain unchanged at `2156df4`, based on
`bd54c49`, and published to `jameinel/sqlair:dollar-placeholders`. That
branch retains the separate, alternative PostgreSQL infrastructure.
Alternative to
[`sqlair-placeholder-style`](../sqlair-placeholder-style/status.md); both
are prepared and proposed together, and the maintainers choose one.

The [prospective PR description](pr.md) is updated for the migrated branch;
no PR has been published. The test explicitly drops its table before the
shared fixture tears down its schema; the fixture's backstop and stale
cleanup command use `CASCADE`. Formatting passed, and on both Go 1.26.11
and 1.27.4, vet, SQLite-only tests, PostgreSQL tests with `-race -p 4
-parallel 4`, and `govulncheck` passed. The functional QA example produced
the expected SQLite and PostgreSQL output. Before migration and after all PostgreSQL
runs, a catalog inventory found no `sqlair_` or `juju-test-` prefixed
schemas; both old and shared cleanup tools also initially found none.
No database or pre-existing schema was dropped.

Real-dqlite validation remains pending. The shared parser rejection is not
integrated: raw `$N` can collide with generated parameters. The shared
decision log and backfill require user confirmation before they are
committed.

Next actions:

- [x] Reserve in the dashboard; record owner/session here.
- [x] Migrate the placeholder layer onto the PostgreSQL infrastructure
  dependency in a new worktree; preserve the original published branch.
- [ ] Integrate the shared parser rejection and complete the remaining
  planned commits.
- [ ] Verify on real dqlite (shared requirements).
- [x] Update `pr.md` locally. PR publication and future pushes need the
  user's approval.

## Scope

Fix [sqlair#189](https://github.com/canonical/sqlair/issues/189) by always
emitting numbered `$N` placeholders, numbered in order of first appearance,
with positional arguments. No public API change. This is the surgical
option: the placeholder text and argument form change for every driver.

Out of scope: placeholder options, `?`/`:name` support, dollar-quoted or
`E''` lexing, empty `IN ()` ([`sqlair-empty-in`](../sqlair-empty-in/status.md)).

## Design

Why first-appearance numbering: SQLite binds `$N` by first appearance and
PostgreSQL by the digit; they agree only when the numbers follow
appearance (shared findings). sqlair already appends arguments in that
order, so the placeholder number equals the argument's position.

1. `internal/expr/querybuilder.go`:
   - `queryBuilder` keeps positional `params []any` instead of
     `sql.Named` values.
   - `addInputs`: append each value, write `$<len(params)>`.
   - `addInsert` / `boundInsertColumn.parameter`: a bulk column appends
     each row's value and writes its new position. A single-value column
     appends once on row 0, records that position on the
     `boundInsertColumn`, and reuses `$<pos>` in later rows. Literals are
     unchanged.
   - Remove `inputAssigner` / `firstInputNum` if they become unused; keep
     the diff otherwise minimal.
   - Comment the first-appearance invariant where numbers are assigned.
2. Output aliases `AS _sqlair_N` are unchanged.
3. Parser: reject bypass `$<digits>` at `Prepare` with a clear error, for
   example "positional parameters are not supported; use $Type.member".
   The parser already skips literals and comments; confirm `$1` inside
   them is still accepted.

## Commits

Commits 1 and 2 must not depend on this branch's placeholder change, so
that `sqlair-placeholder-style` can reuse them unchanged. The first session
to finish them records their hashes in both records; the other
cherry-picks them.

1. `feat(parser): reject positional $N parameters`, with parser tests.
2. `test: run the driver suite against PostgreSQL`: shared helper over the
   backend-neutral `driver_test.go` cases; SQLite run as today; PostgreSQL
   run gated on `SQLAIR_POSTGRESQL_TEST_URL`; test-only pgx requirement;
   `postgres` service in `.github/workflows/go-test.yml`. The PostgreSQL
   run fails until commit 3, so place it after commit 3 in the final
   history.
3. `fix: emit numbered $N placeholders` (fixes #189): the querybuilder
   change; update the 44 `@sqlair_` expectations in
   `internal/expr/expr_test.go`; add one `internal/expr` case for a bulk
   insert mixing bulk and single-value columns, asserting SQL text and
   argument order.
4. `docs: record the placeholder syntax decision`: a new record in
   `docs/decisions/` (next number after the backfill) and an index row.
   It records that sqlair emits `$N` in first-appearance order with
   positional arguments; why (SQLite first-appearance binding,
   PostgreSQL digit binding, `database/sql` ordinal contract); which
   drivers work; why raw `$N` is rejected; and the configurable-style
   alternative it was weighed against. Status: Proposed until merged.

The two shared decision-log commits from
[`sqlair-decision-records`](../sqlair-decision-records/status.md) come
before commit 1.

## Validation

- `go test ./...` with no PostgreSQL URL (SQLite suite).
- `SQLAIR_POSTGRESQL_TEST_URL=... go test ./...` against PostgreSQL 16
  through pgx `stdlib`.
- Real dqlite through Juju (shared requirements), including a bulk insert
  with a single-value column.
- `go vet ./...`; build the docs if the docs toolchain is available
  (`docs/Makefile`).

## Acceptance criteria

- Existing tests pass with only SQL-text expectations updated.
- The driver suite passes on SQLite and PostgreSQL without SQL rewriting.
- A bulk insert with a reused single-value column binds correctly on
  SQLite, dqlite and PostgreSQL.
- `$1` in bypass SQL fails at `Prepare` with a clear error.
- The PR description states the compatibility impact: generated SQL
  changes; drivers without `$N` support (possibly SQL Server through
  `@name`, unverified) would regress; MySQL and Oracle did not work before
  and still do not. It links the alternative PR.
