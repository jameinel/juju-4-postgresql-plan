# 4.1-unitstate-relation-key

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

- Owner/session: `750b409e-35d5-4196-9b64-56ee36a49f56`.
- Branch: `4.1-unitstate-relation-key`.
- Worktree: `.worktrees/4.1-unitstate-relation-key` under the primary checkout.
- Base: `148e09f015fe59237f382b9df1a468c15ee7fa42`.
- Recorded phase: published.

[#23769](https://github.com/juju/juju/pull/23769), head `9a3083a887952914d1457d10084bacfa78bd0c6d`; bot feedback answered, human review required and CI running. Forward-port to main after landing and check migration/export compatibility; reconcile the PoC workaround. The user reports performing functional QA while editing the PR; individual outcomes were not independently verified. At `2f680f1302`, SQLite affected race suites passed; dqlite suites passed except two known #23759 nullable-Boolean machine failures, with changed machine tests passing separately. Scoped lint, txncheck and export regeneration passed. The comment-only follow-up passed transformer package tests, scoped lint, txncheck and gci; default pre-check skipped static analysis. PostgreSQL not tested.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-PLAN.md`:

`4.1-unitstate-relation-key`, #23769 | The original text-binding implementation required rework; **the schema-first rework is implemented** at `2f680f1302`. Review the net patch, integer export/import/recovery and relation deletion paths. Do not revive the superseded normal-path conversions or classify the old PoC workaround as the completed fix.

### Upstream register snapshot

`4.1-unitstate-relation-key` | U21/X23 | [#23769](https://github.com/juju/juju/pull/23769) | `9a3083a887` | open; bot feedback answered, human review required and CI running. User reports performing functional QA; individual outcomes were not independently verified. Forward-port to main and reconcile Stage 5 `1afd3b9434`, a different boundary-only PoC workaround, after landing.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

#### `4.1-unitstate-relation-key` — U21, type mismatch

Published as [#23769](https://github.com/juju/juju/pull/23769), head
`9a3083a887`; continue human review and CI on the existing branch. After
landing, propagate the change to main, checking migration/export compatibility.

The recorded [review reply](https://github.com/juju/juju/pull/23769#issuecomment-6756558330)
answers the legacy-key conversion rationale and defers mainline propagation
until after landing. The transform comment does not require canonical-only
legacy inputs. Human review and current-head CI still need checking.

`setUnitStateRelation` binds Go `int` relation IDs to the `TEXT`
`unit_state_relation.key`, and `GetUnitState` scans it back into `int`.
The user chose to change the column to `INT`, regenerate the 4.1 export
type and convert legacy text keys during import, without adding a foreign
key. Preserve relation filtering and deletion, reject invalid legacy keys,
and validate schema, import/export, recovery and unit-state behavior. The
PoC `1afd3b9434` uses a different, superseded text-binding approach.

## U21 provenance snapshot (2026-11-11 migration)

PoC workaround implemented on Stage 5 (`1afd3b9434`): bind decimal text and parse on read, retaining `TEXT`. This unblocks PostgreSQL but leaves the schema mismatch unresolved. Upstream [#23769](https://github.com/juju/juju/pull/23769), head `9a3083a887`, changes the key to `INT`, regenerates the 4.1 export type and converts legacy text keys during import, without a new foreign key. Bot feedback was answered; human review and CI remain. The user reports performing functional QA; individual outcomes were not independently verified. Forward-port to main and reconcile the differing PoC implementation after landing.
