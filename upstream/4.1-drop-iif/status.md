# 4.1-drop-iif

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Head `fcec930751`: investigate failed revision and charms smoke, upgrade pre-check and aggregate; obtain human review. Complete migration QA and reconcile with the PoC after landing. Preserve the equivalent null-check flags in the connection query and controller views.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-PLAN.md`:

[4.1-drop-iif](https://github.com/jameinel/juju/tree/4.1-drop-iif), [juju/juju#23751](https://github.com/juju/juju/pull/23751) | `fcec930751`: migrating-model test follow-up to the query and schema-view changes | Human review, CI triage and functional QA remain.

From `UPSTREAM-DONE.md`:

`4.1-drop-iif`, [#23751](https://github.com/juju/juju/pull/23751) | X7 | Replace redundant IIF wrappers with equivalent null-check predicates in the connection query and three controller views at `adf9f317a6`.

### Upstream register snapshot

`4.1-drop-iif` | X7 | [#23751](https://github.com/juju/juju/pull/23751) | `fcec930751` | open; revision and charms smoke, upgrade pre-check and aggregate failed; human review and functional QA remain

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### `4.1-drop-iif` — X7

**Published for review (2026-11-04):** [juju/juju#23751](https://github.com/juju/juju/pull/23751)
targets `4.1` from `4.1-drop-iif`, originally with base
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` and head
`adf9f317a6962f5722bb1f4e401db3e9723fd768`; the November 9 head is
`fcec930751`. The query commit
`08b9cc0d7c` and schema-view commit `adf9f317a6` replace all tracked
`IIF` calls with equivalent `IS NOT NULL` predicates. The schema change
affects `v_model_all.is_controller_model` (also projected by `v_model`),
`v_model_state.migrating` and `v_cloud.is_controller_cloud`. The existing
joins and filters are unchanged. Base-DDL editing was chosen because the
local 4.1 tags contain only `v4.1-beta1` and no controller post-patches
are registered. PoC D32 used CASE; reconciliation must choose.

Existing tests cover the connection-query activation/claim combinations
and not-found path, both controller-model and controller-cloud flag values,
and migrating flags for all import phases. No tests were added.
On the schema-view changes, `make run-go-tests` passed with `-race` for
schema and the cloud, model/controller, status/controller,
removal/controller, secretbackend, keyupdater and controllerupgrader state
packages. Schema, cloud, model/controller and status/controller also passed
uncached (`-count=1`). Controller-model view stress completed 48 runs with
zero failures before the bounded timeout. Scoped `golangci-lint`,
repository `txncheck` and `git diff --check` passed; `make pre-check` used
its default, which skips static analysis. PostgreSQL execution and the PR's
functional migration QA have not been performed.

Remaining: triage the current-head failures described above, obtain review,
complete functional QA, and reconcile the PoC after landing. The published PR
description is also saved locally at
[PR-4.1-drop-iif.md](pr.md).

### Historical CI evidence (2026-11-09)

The [revision](https://github.com/juju/juju/actions/runs/42636373516/job/129225558822)
and [charms](https://github.com/juju/juju/actions/runs/42636373516/job/129225558842)
smoke jobs failed after cleanup; the aggregate also failed. The separate
[upgrade pre-check](https://github.com/juju/juju/actions/runs/42636373516/job/129225558535)
reported `no snap found for "juju"`. Do not infer a common root cause or
apply this historical evidence to a later head.
