# 4.1-lease-pin-conflict

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Simon approved on November 9 and suggested removing `database.IsErr*` methods in favor of queries up front; evaluate and respond or defer. GitHub still requires review. Preserve competing-pin, repeated-pin and missing-lease behaviour.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-lease-pin-conflict`, [#23729](https://github.com/juju/juju/pull/23729) | X18 | Resolve competing pins with the explicit lease/entity conflict target. Preserve repeated-pin and missing-lease behaviour; reset the test helper's captured results on transaction retries. This is portability, not a current dqlite defect.

### Upstream register snapshot

`4.1-lease-pin-conflict` | X18 | [#23729](https://github.com/juju/juju/pull/23729) | `c55d5839e5` | open; human approval exists, GitHub review required; evaluate November 9 suggestion

Related scope and shared historical evidence: [4.1-agentbinary-conflict](../4.1-agentbinary-conflict/status.md).

## Lease-pin publication evidence (2026-10-30)

Historical implementation on upstream base `e22f5666d0861d3c84e8465c1ff86fbd23f58f48`;
these checks do not validate later heads.

Session `a646e393-debb-413a-851c-ca78d13c9a23` implemented
`4.1-lease-pin-conflict` in `../4.1-lease-pin-conflict`, restricted to
`domain/lease/state`. Initial published commit `373ab4f422` opened
juju/juju#23729 using [the local description](../4.1-lease-pin-conflict/pr.md).
`make run-go-tests TEST_PACKAGES=./domain/lease/state/...` passed with
`-race -count=3` and `-race -count=20 -run TestStateSuite/TestPin`; lint
was clean. Regressions also passed with the old SQL because SQLite keeps
the transaction usable after the caught constraint failure. This is
portability, not a current dqlite defect; no production timing change
belongs in this branch. Later published follow-ups moved its head to
`c55d5839e5`; the historical checks above do not establish its validation.
