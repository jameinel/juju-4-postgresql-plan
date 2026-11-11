# 4.1-sql-standard-syntax

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-27; merge `302b44cc2c`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-sql-standard-syntax`, #23695; `4.1-sql-test-literals`, #23696 | Aligned: equality/join formulation and literal quoting are fixed in SQL itself, including production paths. No D55-driven follow-up indicated.

From `UPSTREAM-DONE.md`:

`4.1-sql-standard-syntax`, #23695 | X37/X38 | Standard equality and an explicit CROSS JOIN to the one-row fulfilment aggregate. Cover unused/shared-key deletion and zero-fulfilment detach info. A third commit changes storage lifecycle trigger equality.

### Upstream register snapshot

`4.1-sql-standard-syntax` | X37, X38 | [#23695](https://github.com/juju/juju/pull/23695) | `13003d8680` | merged 2026-10-27; merge `302b44cc2c`
