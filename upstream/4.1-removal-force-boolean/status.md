# 4.1-removal-force-boolean

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-27; merge `9ea82c03bd`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-removal-force-boolean`, #23690 (U9) | Aligned: BOOLEAN storage, bool exports and validated legacy 0/1 import conversion. Do not undo this by converting normal writes to integers.

From `UPSTREAM-DONE.md`:

`4.1-removal-force-boolean`, #23690 | U9 | Match boolean writers/readers with BOOLEAN storage and bool exports; convert legacy 0/1 and reject other integers.

### Upstream register snapshot

`4.1-removal-force-boolean` | U9 | [#23690](https://github.com/juju/juju/pull/23690) | `f39572e43b` | merged 2026-10-27; merge `9ea82c03bd`

## U9 provenance snapshot (2026-11-11 migration)

Fixed, Stage 6, `fix(schema): declare the removal force flag as boolean`. `BOOLEAN NOT NULL DEFAULT FALSE` in place; the 4.1 export type becomes `bool` and the 4.0.12 import delta converts 0 / 1 and rejects other values. Upstream, drop the `domain/schema/postgresql/` paths from the commit.
