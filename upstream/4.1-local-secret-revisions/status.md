# 4.1-local-secret-revisions

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-11-04; merge `edb2aef5b9`; reconciled into PoC 2026-11-05

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-local-secret-revisions`, [#23720](https://github.com/juju/juju/pull/23720) | U11 follow-up | Independently select latest local consumed-secret revisions in two query methods, with empty/multiple-secret/query-round-trip regressions and redundant deduplication removed. Merged 2026-11-04; the five-query latest-revision task remains separate.

### Upstream register snapshot

`4.1-local-secret-revisions` | U11 follow-up | [#23720](https://github.com/juju/juju/pull/23720) | `ee07d7ccb5` | merged 2026-11-04; merge `edb2aef5b9`; reconciled into PoC 2026-11-05

Related scope and shared historical evidence: [4.1-crossmodel-secret-revisions](../4.1-crossmodel-secret-revisions/status.md).
