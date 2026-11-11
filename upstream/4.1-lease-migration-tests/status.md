# 4.1-lease-migration-tests

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-27; merge `146df46e46`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-lease-migration-tests`, #23686 (U4) | The silent setup-error defect is fixed, but the merged helper still uses `datetime('now', ?)` with SQLite modifiers. It does not bind computed timestamps as earlier documentation claimed. An independent fixture-portability follow-up is [queued at low priority](../lease-migration-fixture-timestamps/status.md#todo-lease-migration-fixture-timestamps-low-priority); no storage-type rewrite is indicated.

From `UPSTREAM-DONE.md`:

`4.1-lease-migration-tests`, #23686 | U4 | Assert seed success and seed held/expired leases with SQLite datetime expressions so tests exercise actual rows rather than silently passing on empty setup. Computed timestamp binding remains a possible separate portability follow-up, not part of the merged implementation.

### Upstream register snapshot

`4.1-lease-migration-tests` | U4 | [#23686](https://github.com/juju/juju/pull/23686) | `7e3e903650` | merged 2026-10-27; merge `146df46e46`

## U4 provenance snapshot (2026-11-11 migration)

PoC fixed on Stage 6 with bound timestamps. Merged #23686 fixes setup error checking but retains SQLite `datetime('now', ?)` fixtures; a separate, low-priority portability follow-up is [queued in UPSTREAM-PLAN.md](../lease-migration-fixture-timestamps/status.md#todo-lease-migration-fixture-timestamps-low-priority). Do not describe the merged implementation as already binding computed timestamps.
