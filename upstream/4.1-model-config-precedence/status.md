# 4.1-model-config-precedence

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Review follow-ups are published; reported checks have no failures or pending jobs. Obtain remaining approval. Preserve authoritative agent-table values, import filtering and one effective value per key.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-model-config-precedence`, #23683 | U7 | Exclude agent stream/version from competing config rows so authoritative `agent_version` values win regardless of UNION sorting or row order.

### Upstream register snapshot

`4.1-model-config-precedence` | U7 | [#23683](https://github.com/juju/juju/pull/23683) | `f50dfe4257` | open; no failing/pending reported checks; approval required

## U7 provenance snapshot (2026-11-11 migration)

Fixed, Stage 6, `fix(schema): prefer agent_version values in v_model_config`. The model branch excludes `agent-stream` and `agent-version` and the view uses `UNION ALL`; `TestModelConfigAgentStreamAndVersionInConfig` fails on dqlite without it.
