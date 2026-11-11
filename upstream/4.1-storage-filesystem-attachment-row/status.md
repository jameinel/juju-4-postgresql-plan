# 4.1-storage-filesystem-attachment-row

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

- Owner/session: `fc0b395c-35d9-4277-ba1d-3bc4779db286`.
- Branch: `4.1-storage-filesystem-attachment-row`.
- Worktree: `.worktrees/4.1-storage-filesystem-attachment-row` under the primary checkout.
- Base: `148e09f015fe59237f382b9df1a468c15ee7fa42`.
- Recorded phase: blocked.

[#23766](https://github.com/juju/juju/pull/23766), head `2bfbdc7d6077364266c0756ad9898dd8a830d8ee`; merge hold remains pending acceptance of duplicate rejection and its source-repair tradeoff. Retain ownership; storage and migration QA remain pending.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-PLAN.md`:

`4.1-storage-filesystem-attachment-row`, #23766 | **Hold merge; decision and QA required.** Head `2bfbdc7d60` replaces lowest-UUID selection with filesystem/node uniqueness and rejects legacy duplicates with diagnostics. Decide whether the migration and source-repair tradeoff is acceptable; no general safe repair is verified. Complete storage and migration QA before lifting the hold. Do not silently discard rows.

### Upstream register snapshot

`4.1-storage-filesystem-attachment-row` | U22/X16 (latent bug) | [#23766](https://github.com/juju/juju/pull/23766) | `65dbcae6a9` | open; user-requested local merge hold, invariant investigation and rework before merge; retain owner and do not settle semantics through UUID ordering

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

#### `4.1-storage-filesystem-attachment-row` — U22, latent bug (priority)

Published as [#23766](https://github.com/juju/juju/pull/23766); continue
on the existing branch, but **hold merge pending a decision and QA**.
Head `2bfbdc7d60` enforces filesystem/node uniqueness and rejects legacy
duplicates; the tradeoff and safe source repair remain unaccepted and
unverified. Storage and migration QA remain pending.

`domain/storage/state/instance.go` `GetStorageInstanceInfo` joins every
filesystem attachment on the unit's net node and collapses the fan-out
with `GROUP BY sa.uuid`. The schema does not forbid two filesystem
attachments for the same filesystem and net node, so dqlite returns an
arbitrary mount point. The earlier implementation picked the lowest
attachment UUID via a correlated subquery, dropped the `GROUP BY`, and moved
the `ORDER BY` out of the `FROM` subquery in both the attachment and the
block-device link queries. The regression used two filesystem attachments on the
same node; the PoC test failed 7 of 10 runs against the old query.
The current head uses a uniqueness index instead and prechecks duplicate
imports. It may reject a second attachment to colocated units and block
migration of affected 4.0 models; no general repair is verified. Do not
silently deduplicate data or lift the hold without approval. PoC: `640d3bcabf`.
