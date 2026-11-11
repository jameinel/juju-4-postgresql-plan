# lease-migration-fixture-timestamps

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Unclaimed follow-up; this directory is a task identifier, not evidence
that a branch or worktree has been created. Confirm scope and reserve
the task in the dashboard before implementation.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

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
