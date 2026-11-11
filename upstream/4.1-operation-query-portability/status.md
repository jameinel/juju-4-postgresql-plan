# 4.1-operation-query-portability

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Human approval exists; await the remaining GitHub review gate. Commit identity/message and rebase requests were addressed. Preserve filtering and the reviewed NULL-as-zero column-size fix, which changes the old whole-row-ignored estimate.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-operation-query-portability`, #23693 | X34/X33 | Portable boolean comparisons and text-size expressions, plus the reviewed fix that treats each NULL column as zero instead of discarding the entire row from the estimate. Filtering is preserved; size/pruning decisions for nullable rows intentionally change. Text size is not stored bytes.

### Upstream register snapshot

`4.1-operation-query-portability` | X34, X33 | [#23693](https://github.com/juju/juju/pull/23693) | `0fb2eb7194` | open; current macOS and aggregate checks passed; human approval exists, GitHub review required
