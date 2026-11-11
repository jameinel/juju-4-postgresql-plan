# 4.1-cmr-ingress-order

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

- Owner/session: `87b89739-8e7d-4e6e-a852-8cf8fa0e4d61`.
- Branch: `4.1-cmr-ingress-order`.
- Worktree: `.worktrees/4.1-cmr-ingress-order` under the primary checkout.
- Base: `bd970f693c8e5060973016fe9059108570fa5bd3`.
- Recorded phase: published.

[#23805](https://github.com/juju/juju/pull/23805), head `1f2411ad8c013246780abcc0e1077db179d51197`; review required. CI is running; arm64 snap build, migration, upgrade pre-check and several smoke jobs failed, cause uninvestigated. SQLite and real-dqlite affected race suites, scoped lint, txncheck, gci and diff checks passed; pre-check skipped static analysis. PostgreSQL and live controller QA not run. Triage CI, obtain maintainer confirmation of the ordering contract and complete functional QA.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### `4.1-cmr-ingress-order` — X17, explicit CIDR ordering

**Status:** guaranteed lexical ordering chosen by the user on 2026-11-10
and implemented locally in the PoC and upstream
[#23805](https://github.com/juju/juju/pull/23805). Upstream intent remains
unconfirmed; review, CI and functional QA are pending.

`GetRelationNetworkIngress` in
`domain/crossmodelrelation/state/model/relation_network.go` selects CIDRs
without ordering, while `TestGetRelationNetworkIngressSortedOrder` explicitly
expects sorted output. The primary key is `(relation_uuid, cidr)`, which can
yield incidental index order on SQLite; the firewaller consumes a set.

The local decision is to preserve the sorted behavior explicitly with
`ORDER BY cidr`, document lexical ordering in the state/service contracts,
and retain coverage distinguishing lexical from numeric address ordering.
Do not extend this decision to sibling egress queries.

**Required upstream review question:** was sorted output strictly intended
as an API contract, or was the test capturing incidental SQLite ordering?
When preparing the upstream fix, explain the local decision and ask
maintainers to confirm the intended contract. Record their response and
report it back to the user; do not present the historical intent as settled.
If upstream intends an unordered collection instead, revisit the local
decision with the user before changing direction.
