# 4.1-lxd-default-bases

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-11-11 as `bd970f693c`; PoC rebase includes the fix but validation remains incomplete

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-lxd-default-bases`, [#23756](https://github.com/juju/juju/pull/23756) | U13 follow-up, test diagnostics and dqlite CI | Preserve SSH-key setup diagnostics and failure status; avoid corrupted retryable dqlite `RETURNING` errors in keymanager, with a sentinel and broader real-dqlite CI. Merged 2026-11-11.

### Upstream register snapshot

`4.1-lxd-default-bases` | U13 follow-up, test diagnostics, dqlite CI | [#23756](https://github.com/juju/juju/pull/23756) | `63a93201b5` | merged 2026-11-11 as `bd970f693c`; PoC rebase includes the fix but validation remains incomplete

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## LXD diagnostics and keymanager retryability

[#23756](https://github.com/juju/juju/pull/23756) merged on 2026-11-11
at 00:01:51 UTC as `bd970f693c8e5060973016fe9059108570fa5bd3`,
with final head `63a93201b565ab68f2167aef8736697ccacaa839`. SSH-key
setup now reports captured errors and preserves failure status under
`set -e`; integration-test changes trigger smoke jobs. The keymanager
uses an insert and same-transaction unique-key lookup to preserve retryable
errors while retaining unknown-algorithm rejection. A real-dqlite sentinel
tracks the upstream `RETURNING` fix; the SQLite stale-snapshot regression
checks WAL mode. Real-dqlite CI covers domain suites except the two packages
addressed by follow-up #23759.

Pedramktb and SimonRichardson approved; Van Cleef's warning and three
suggestions were addressed. The author reported that all five shell QA
harness cases, Bash syntax and ShellCheck passed on the November 5
follow-up; that evidence predates later keymanager and CI commits.
At the final head GitHub reported successful default-bases smoke, unit
tests and aggregate, with five checks skipped. Repeated live LXD deploy QA
was not recorded. The PoC rebase includes the merged keymanager fix, but
its residual test work and validation are incomplete; see the
[reconciliation record](../../UPSTREAM-FIXES.md#poc-reconciliation-onto-bd970f693c-2026-11-11).
Dependency-level driver fixes remain separate follow-up work.
