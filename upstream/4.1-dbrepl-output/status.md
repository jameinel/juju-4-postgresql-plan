# 4.1-dbrepl-output

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Confirm the current owner and state before acting; this migration does
not assign ownership or authorize changes to an existing worktree.

Recorded state: juju/juju#23757 merged on 2026-11-09 at head `0583a1972c`.
The shared-renderer task is complete upstream; live REPL QA was not recorded.
Dbrepl remains outside the PostgreSQL PoC scope.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-dbrepl-output`, [#23757](https://github.com/juju/juju/pull/23757) | X16 follow-up | Share change-log/change-stream rendering without changing either query or output. Merged 2026-11-09.

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Dbrepl output refactor

Session `1fedb56c-3ba7-4099-8914-c6726b9cd337` used branch
`4.1-dbrepl-output` and worktree `juju-4.1-dbrepl-output`, based on
upstream 4.1 `08fafe7f49ee39bed52c7288e86be0df6af4adcf`. Commit
`0583a1972cc6ecff30c36d8653c11066162c2770` shares change-log and
change-stream row scanning and tabwriter output without changing either
query, arguments, output, resource handling or error reporting. Tests cover
repeated keys, zero ID, empty and identical output, and query-error logging.

[#23757](https://github.com/juju/juju/pull/23757) merged on 2026-11-09 at
17:39:23 UTC as `775513d9df09cebcfada49bf1bfcadc975b134df`, with final
head `0583a1972cc6ecff30c36d8653c11066162c2770`. Gvantsats and
SimonRichardson approved; Van Cleef's review was advisory. The final-head
checks reported successes and skips, not a pass for skipped jobs.

Historical validation on that commit: the dbrepl package race suite passed
on SQLite and real dqlite; scoped lint, `txncheck`, `gci`, `gofmt` and
`git diff --check` passed. `make pre-check` skipped static analysis.
Real-dqlite stress completed 20 runs without failures before the bounded
timeout, with two runs still active. Live REPL QA, PostgreSQL execution,
the full repository suite and full static analysis were not run. Dbrepl is
outside the PostgreSQL PoC scope.
