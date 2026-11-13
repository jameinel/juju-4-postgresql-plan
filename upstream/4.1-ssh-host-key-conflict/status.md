# 4.1-ssh-host-key-conflict

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

- Owner/session: `dbd7747e-8c2d-4b78-8994-1cd20589b8ab` (takeover
 authorised by the user on 2026-11-12).
- Branch: `4.1-ssh-host-key-conflict`.
- Worktree: `.worktrees/4.1-ssh-host-key-conflict` under the primary checkout.
- Base: `bd970f693c8e5060973016fe9059108570fa5bd3`.
- PR: [#23807](https://github.com/juju/juju/pull/23807); live head rechecked
 on 2026-11-13 20:51 UTC: `b2c4c9198919195e76a90d38f0882c0c300beefb`.
- Phase: follow-up pushed and approved PR description and review response
 published. Simon approved on 2026-11-13, but GitHub still reports
 `REVIEW_REQUIRED` and aggregate checks `FAILURE`. The remaining review
 gate, CI triage and live QA remain outstanding.

Simon requested returning the ordinary query error instead of special-casing
`ErrNoRows`; the follow-up does so for both paths. Van Cleef's rationale
comments were added; its shared-helper suggestion was considered and declined
because it would require nine arguments and a record callback. Removal and
recovery do have key-row deletion paths, contrary to its review claim.
Drafts: [PR description](pr.md), [Simon reply](reply-simon.md),
[Van Cleef reply](reply-van-cleef.md). The user has
[replied to Simon](https://github.com/juju/juju/pull/23807#discussion_r4558701852);
the thread remains unresolved. Do not post the draft reply as a duplicate.
The [Van Cleef response](https://github.com/juju/juju/pull/23807#issuecomment-6841485809)
and the prepared PR description were published and verified.

At the local follow-up's tree on 2026-11-12, SQLite and real-dqlite SSH model
state suites passed with `-race`; each backend passed 10 stress runs of both
concurrent host-key tests. Scoped `golangci-lint`, compiled `txncheck`, `gci`,
diff checks and `STATIC_ANALYSIS=1 make pre-check` completed successfully.
The latter skipped licence analysis (`go-licenses` missing), `shfmt` and
`govulncheck`; these are not passes. A subsequent direct run of the repository's
`go-licenses check ./...` command passed after the user installed the tool;
it warned that non-Go code could not be inspected for further dependencies.
A direct `shfmt -l -s` check with version 3.12.0 reported 30 existing shell
files; CI uses 3.7.0. The repository wrapper appears to filter for temporary
files instead of the intended source files. The user chose to leave
`govulncheck` to CI. CI reported `GO-2026-6859`,
Snap Store failures and a JAAS bootstrap API connection refusal at the earlier
head `e00ed96510`; checks for `b2c4c91989` were queued or in progress at
that snapshot. The current aggregate failure is recorded above.
PostgreSQL and live functional QA were not run. Follow up CI and functional QA
including Kubernetes, then reconcile PoC `2f55c00439` after landing.

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
