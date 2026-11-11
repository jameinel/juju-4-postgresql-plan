# 4.1-changestream-latest-row

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-30; merge `ece31c289c`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-changestream-latest-row`, #23691; `4.1-dbrepl-latest-row`, #23730; `4.1-local-secret-revisions`, #23720 | Aligned: explicit latest-ID/revision selection and joins/filters replace bare aggregate columns. No D55-driven redo. U19's other secret queries remain separate work; #23720 did not claim to fix them all.

From `UPSTREAM-DONE.md`:

`4.1-changestream-latest-row`, #23691 | X16 | Explicitly select the latest change ID and join to its row, avoiding ungrouped columns beside MAX. This does not fix the separate dbrepl query.

### Upstream register snapshot

`4.1-changestream-latest-row` | X16 | [#23691](https://github.com/juju/juju/pull/23691) | `9e28053164` | merged 2026-10-30; merge `ece31c289c`
