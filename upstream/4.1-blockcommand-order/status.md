# 4.1-blockcommand-order

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-27; merge `b36f88d99c`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-blockcommand-order`, #23692 | Aligned: explicit block-type ordering replaces implicit rowid ordering. This is the agreed ordering change, not a conversion workaround.

From `UPSTREAM-DONE.md`:

`4.1-blockcommand-order`, #23692 | U10/X31 | Replace implicit rowid order with explicit block-type order. This is an agreed observable ordering change, not a syntax-only rewrite.

### Upstream register snapshot

`4.1-blockcommand-order` | U10/X31 | [#23692](https://github.com/juju/juju/pull/23692) | `495768b0aa` | merged 2026-10-27; merge `b36f88d99c`

## U10 provenance snapshot (2026-11-11 migration)

Fixed, Stage 5, `fix(blockcommand): order blocks by type instead of rowid` (D31). dqlite order changes to type order; `TestGetBlocksOrderedByType` fails with `rowid`.
