# 4.1-lease-migration-timestamps

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Reserved 2026-11-18 by session `3affe058-7602-478e-8c64-1de89f410003`
with coordinator approval.

- Branch: `4.1-lease-migration-timestamps`.
- Worktree: `juju/.worktrees/4.1-lease-migration-timestamps`.
- Upstream 4.1 base: `d380e5e4fba4c9918a0774638ffa3138aeece73e`.
- Local head: `d7e0000217c4c9f962827f92a5225baaec9a7253`.
- Fork branch: `jameinel/juju:4.1-lease-migration-timestamps`, verified at
  local head after the approved push on 2026-11-18.
- PR: [juju/juju#23884](https://github.com/juju/juju/pull/23884), open at
  the verified local head, created with approval on 2026-11-18.
- Next: await review and CI; investigate any feedback or failures. PoC
  reconciliation remains a separate coordinator/integration-owner task.
- Duplicate check: the upstream test file is unchanged from the local 4.1
  checkout; no matching local branch or open lease-migration PR was found.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### TODO: lease migration fixture timestamps (low priority)

**Status: published and locally validated; review and CI pending.** Independent follow-up to merged
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

## Implementation and validation (2026-11-18)

The fixture binds UTC whole-second text computed from one local clock
reading per insertion, matching the representation written by the existing
SQLite lease state. This removes the fixture's SQLite-specific datetime
expressions; the production query, schema and lease semantics are unchanged.

At head `010a29babaadb8fed62050f69f21d7aa2869185a`:

- `go test ./domain/lease/state -run '^TestMigrationSuite$' -count=1 -timeout 120s`
  passed on SQLite.
- `go test ./domain/lease/state -count=1 -timeout 120s` passed on SQLite.
- `timeout 180s make run-go-tests TEST_PACKAGES=./domain/lease/state
  TEST_ARGS='-v -race -count=1 -timeout 150s'
  TEST_FILTER='^TestMigrationSuite$'` passed with real dqlite.
- `timeout 180s make run-go-tests TEST_PACKAGES=./domain/lease/state
  TEST_ARGS='-race -count=1 -timeout 150s'` passed with real dqlite.
- `gci` on the changed file, scoped `golangci-lint run -c .golangci.yml
  ./domain/lease/state/...`, `txncheck "$PWD"` and `git diff --check` passed.

No PostgreSQL execution or full pre-check was run. Live migration QA is not
applicable to this test-only change, as agreed for the PR.
The existing production `date('now')` cutoff is a separately scoped
question and remains unchanged.

## Database clock review (2026-11-18)

The user requested considering database time instead of the host clock and
asked whether the inserts can use `CURRENT_TIMESTAMP` with an offset directly.
SQLite rejected `CURRENT_TIMESTAMP + INTERVAL '1' DAY` with a syntax error;
`CURRENT_TIMESTAMP + 86400` performed numeric coercion rather than timestamp
arithmetic. SQLite's `datetime(CURRENT_TIMESTAMP, '+1 day')` works, but
retains the SQLite-specific function this task is intended to remove.
The user chose the host-clock implementation with an explanatory comment.
The suite creates its own local database, so Go and the database use the
same host clock; the assertions need held or expired values, not precise
timestamps. The original offsets are retained. Abrupt clock changes between
fixture insertion and the query are not simulated.

At revised head `d7e0000217c4c9f962827f92a5225baaec9a7253`, the focused
`TestMigrationSuite` passed again on SQLite and real dqlite (the latter
with `-race`), and `gci` and `git diff --check` passed. The full package
and lint checks above predate the comment-only follow-up.

## Publication (2026-11-18)

The approved fork push and PR creation were verified at
`d7e0000217c4c9f962827f92a5225baaec9a7253`. The published body matches
`pr.md` byte for byte. At the initial observation, GitHub required review
and checks were queued or running; some checks were skipped.
