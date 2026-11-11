# 4.1-keymanager-insert-id

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-27; merge `f0adf0150d`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-keymanager-insert-id`, #23684 (U13) | Aligned: tie the ID to the actual insert and reject an unknown algorithm instead of trusting stale LastInsertId state. A follow-up is reasonable and already open as #23756 for the separate dqlite RETURNING error-response defect. Keep the original correctness guard.

From `UPSTREAM-DONE.md`:

`4.1-keymanager-insert-id`, #23684 | U13 | Obtain the key ID from the inserted row. A zero-row insert for an unknown algorithm must not reuse a prior `LastInsertId` and associate an unrelated key.

### Upstream register snapshot

`4.1-keymanager-insert-id` | U13 | [#23684](https://github.com/juju/juju/pull/23684) | `d490585ad7` | merged 2026-10-27; merge `f0adf0150d`

## U13 provenance snapshot (2026-11-11 migration)

Fixed, Stage 5, `fix(keymanager): return new public key ids from the insert` (D36, DIALECT.md X35). An unknown algorithm returns an error; new test.
