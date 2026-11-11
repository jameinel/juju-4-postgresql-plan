# 4.1-controller-export-null

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Independent head `2e12c3150a`. Evaluate Van Cleef's three suggestions; investigate failed localhost smoke and aggregate, obtain human review and complete disposable-controller backup QA. Preserve nullable Boolean and timestamp values without changing the export format.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

### Upstream register snapshot

`4.1-controller-export-null` | Controller backup NULL fidelity | [#23765](https://github.com/juju/juju/pull/23765) | `2e12c3150a` | open; evaluate bot suggestions; localhost smoke and aggregate failed; human review and backup QA remain

Related scope and shared historical evidence: [4.1-dqlite-nullable-boolean](../4.1-dqlite-nullable-boolean/status.md).

## Scope and acceptance

Keep this independent from the nullable-Boolean domain/CI workaround.
Preserve NULL Boolean and datetime/timestamp pointers in controller exports,
using `IS NULL` markers following model export, without changing the backup
format. Reset the captured payload on transaction retry.

Cover distinct NULL/false/true flags and NULL/set timestamps on SQLite and
real dqlite. Retain export-generator tests and regeneration without drift.
Review the nullable-field coverage and generator-drift suggestions, triage
CI and complete the disposable-controller backup QA. Lease rows are exported
but not restored; do not claim a restore round trip for them.

The shared driver investigation later recorded manual QA for local head
`3b733c237b`; the older review snapshot above names `2e12c3150a`. Recheck the
actual local/published heads before acting, and keep evidence revision-specific.
When the bundled dqlite includes canonical/dqlite#883, review the temporary
export NULL markers with the related branch, validating Boolean and timestamp
fidelity rather than simply removing the workaround.

### Historical CI evidence (2026-11-09)

The [localhost LXD smoke job](https://github.com/juju/juju/actions/runs/42258592467/job/128230348819)
failed after cleanup; its aggregate also failed. This does not establish
the cause or describe later heads.
