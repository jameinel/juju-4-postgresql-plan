# 4.1-ssh-host-key-conflict

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

- Owner/session: `d36531ca-29ec-454e-af43-9c0f627b8c93`.
- Branch: `4.1-ssh-host-key-conflict`.
- Worktree: `.worktrees/4.1-ssh-host-key-conflict` under the primary checkout.
- Base: `bd970f693c8e5060973016fe9059108570fa5bd3`.
- Recorded phase: published.

[#23807](https://github.com/juju/juju/pull/23807), head `e00ed96510b49abd0a1d78efc1d99e20cf6495f7`; human review required and CI pending. SQLite and real-dqlite affected race suites and 10 concurrent stress runs per backend passed, as did scoped lint, txncheck and diff checks. Default pre-check skipped static analysis. Initial focused dqlite invocation failed on shell quoting; the complete suite passed. PostgreSQL and live QA not run. Complete review, CI and functional QA, including the Kubernetes unit path; reconcile PoC `2f55c00439` after landing.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### `4.1-ssh-host-key-conflict` — U24, conflict-safe host-key creation

**Status:** PoC Stage 5 `2f55c00439` implements the approved approach.
The independent upstream breakout is [#23807](https://github.com/juju/juju/pull/23807),
head `e00ed96510`; see the current-state section above for validation and next actions.

`domain/ssh/state/model/state.go` implements
`EnsureMachineVirtualHostKeyByMachineName` and
`EnsureUnitVirtualHostKeyByUnitName`. Each must persist the proposed key only
when no key exists, otherwise return the existing persisted key. The service
can generate competing candidates after concurrent absence checks. The
current insert/catch-constraint/read formulation depends on a transaction
remaining usable after a failed statement.

Use `INSERT ... ON CONFLICT (machine_uuid) DO NOTHING` and the corresponding
`unit_uuid` form, followed by reading the persisted key in the transaction.
Preserve the transaction runner's retry semantics. Never overwrite the
winning key or algorithm, suppress unrelated errors, or substitute a
check-then-insert sequence that leaves the creation race unresolved.

Cover both machine and unit paths: initial creation, existing keys with
different proposed keys/algorithms, competing creators returning the same
persisted key, missing entities and other constraint/error paths. Validate
SQLite and real dqlite, with race detection and bounded stress for concurrent
coverage. PostgreSQL exposed the aborted-transaction failure, but PostgreSQL
runtime support or test infrastructure is not part of this upstream patch.
Keep this concern independent of other SSH changes; record branch, source
commits, validation and eventual PoC reconciliation separately.

## U24 provenance snapshot (2026-11-11 migration)

Fixed on PoC Stage 5 as `2f55c00439`: targeted conflict-safe inserts followed by a read of the persisted key. Focused PostgreSQL and real-dqlite checks and concurrent stress passed at `09=85e4e6b122`; see Stage 5. Independent upstream [#23807](https://github.com/juju/juju/pull/23807), head `e00ed96510`, awaits review, CI and live QA; reconcile after landing.
