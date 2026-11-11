# 4.1-application-exposure-scope

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Reported checks have no failures or pending jobs; obtain approval. Rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-application-exposure-scope`, #23723 | U15, exposure | Separate application-wide wildcard exposure from endpoint-specific rows and require endpoint IDs in endpoint tables.

### Upstream register snapshot

`4.1-application-exposure-scope` | U15 (exposure + combined guard) | [#23723](https://github.com/juju/juju/pull/23723) | `4e66f7c449` | open; no failing/pending reported checks; approval required; rebase/regenerate after charm lands; cherry-picked onto PoC Stage 5 (D43)

Related scope and shared historical evidence: [4.1-nullable-primary-keys](../4.1-nullable-primary-keys/status.md).
