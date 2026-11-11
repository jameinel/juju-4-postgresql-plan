# 4.1-controller-node-id-text

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-27; merge `6e98deb0e0`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-controller-node-id-text`, #23685 (U3/U5) | Aligned exception: dqlite node IDs span uint64, beyond signed SQLite integers. Decimal TEXT is intentional storage; formatting it at the boundary is correct. Do not propose an INT-schema follow-up based on U21's different domain contract.

From `UPSTREAM-DONE.md`:

`4.1-controller-node-id-text`, #23685 | U3/U5 | Bind decimal text to the TEXT node-ID column, including high-bit and max-uint64 values; correct changestream and upgrade fixtures. Do not narrow IDs to signed INT.

### Upstream register snapshot

`4.1-controller-node-id-text` | U3, U5 | [#23685](https://github.com/juju/juju/pull/23685) | `55ee33f048` | merged 2026-10-27; merge `6e98deb0e0`

## U3 provenance snapshot (2026-11-11 migration)

Fixed, Stage 6, `fix(database): bind controller node id as text`.

## U5 provenance snapshot (2026-11-11 migration)

changestream fixed, Stage 6, `test(changestream): insert controller node ids as text`; upgrade fixed upstream in juju/juju#23685. Both landed; PoC commit dropped 2026-11-05.
