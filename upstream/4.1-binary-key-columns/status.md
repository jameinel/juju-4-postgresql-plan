# 4.1-binary-key-columns

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-27; merge `fa1743d544`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-binary-key-columns`, #23682 (U8) | Aligned: raw key bytes become BLOBs and the obsolete TEXT storage exceptions are removed. Matching export/recovery changes are necessary, not a workaround for the old schema. No D55-driven follow-up indicated.

From `UPSTREAM-DONE.md`:

`4.1-binary-key-columns`, #23682 | U8 | Store activation, bakery and macaroon root keys as BLOBs, with matching export/recovery and legacy compatibility. Raw random bytes are not text.

### Upstream register snapshot

`4.1-binary-key-columns` | U8 | [#23682](https://github.com/juju/juju/pull/23682) | `de69a748a6` | merged 2026-10-27; merge `fa1743d544`

## U8 provenance snapshot (2026-11-11 migration)

Fixed, Stage 6, `fix(schema): store random key bytes in blob columns`. The columns become `BLOB` in place; the 4.1 controller export types become `[]byte`. Upstream, drop the `domain/schema/postgresql/` paths from the commit.
