# 4.1-agentbinary-conflict

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

[PR #23728](https://github.com/juju/juju/pull/23728), branch `4.1-agentbinary-conflict`, merged 2026-11-06 as `32be283d02` at head `2618d146da` with shared registration state. Both paths use `ON CONFLICT (version, architecture_id) DO NOTHING` and compare the retained registration, preserving `AlreadyExists` / `AgentBinaryImmutable`, idempotency and transaction usability. Earlier commits were cherry-picked onto `pg-poc/05-queries` 2026-11-05 (D43, identical patch IDs): `323807680f`→`fc66010f1a`, `6a66e5ed1b`→`c34b912abb`. These predate the shared-state refactor; reconcile the merged result. Default-backend tests pass on the recorded PoC stack; PostgreSQL and bootstrap not yet re-run.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-agentbinary-conflict`, [#23728](https://github.com/juju/juju/pull/23728) | U16/X18 | Use `ON CONFLICT (version, architecture_id) DO NOTHING` and compare retained registration in both databases, preserving idempotency, immutability and transaction usability. Merged 2026-11-06.

From `UPSTREAM-FIXES.md`:

`4.1-agentbinary-conflict` | U16/X18 | [#23728](https://github.com/juju/juju/pull/23728) | `2618d146da` | merged 2026-11-06 as `32be283d02`; PoC cherry-picks predate the shared-state refactor and require reconciliation

### Upstream register snapshot

U16 | latent | `domain/agentbinary/state/{controller,model}/state.go` `RegisterAgentBinary` | On a primary key conflict it runs a follow-up `SELECT` in the same transaction. SQLite keeps the transaction usable after the failed statement; PostgreSQL does not (DIALECT.md X18, 25P02 "current transaction is aborted"). The caller's `AlreadyExists` / `AgentBinaryImmutable` handling is never reached. **Found in Stage 9 (2026-10-28):** retries after the U15 failure re-register the binary and fail with 25P02. | [PR #23728](https://github.com/juju/juju/pull/23728), branch `4.1-agentbinary-conflict`, merged 2026-11-06 as `32be283d02` at head `2618d146da` with shared registration state. Both paths use `ON CONFLICT (version, architecture_id) DO NOTHING` and compare the retained registration, preserving `AlreadyExists` / `AgentBinaryImmutable`, idempotency and transaction usability. Earlier commits were cherry-picked onto `pg-poc/05-queries` 2026-11-05 (D43, identical patch IDs): `323807680f`→`fc66010f1a`, `6a66e5ed1b`→`c34b912abb`. These predate the shared-state refactor; reconcile the merged result. Default-backend tests pass on the recorded PoC stack; PostgreSQL and bootstrap not yet re-run.

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Agent-binary and lease-pin publication evidence

These are historical 2026-10-30 implementation checks, not new validation
of current PR heads. Both independent branches started at upstream
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` with shared `_deps`.

Session `2b06495a-18f9-4418-ba5c-b2e8fd908dd1` implemented
`4.1-agentbinary-conflict` in `../4.1-agentbinary-conflict`, restricted to
`domain/agentbinary/state/{controller,model}`. Implementation
`323807680f7a5249f65d2d7378c0c2158cb3edee` and comment-only follow-up
`6a66e5ed1be4cc0bde5fee269b6da1f87da56440` were pushed after approval
as juju/juju#23728. The revised description was approved and verified
against [the local text](../../archive/pr-descriptions/PR-4.1-agentbinary-conflict.md).

Conflict-safe inserts and affected-row checks preserve error sentinels and
immutability. New tests cover same-transaction writes after both conflict
cases, original mapping retention and probe-write rollback. Validation:

- `make run-go-tests` with
  `TEST_PACKAGES='./domain/agentbinary/state/controller ./domain/agentbinary/state/model'`,
  `TEST_FILTER='Test.*StateSuite/TestAdd'`, `TEST_ARGS='-p 2 -race'`: passed.
- The same command without `TEST_FILTER`: both full state race suites passed.
- Each package compiled with `make run-go-tests`,
  `TEST_ARGS='-p 2 -race -c -o <session-binary>'`, then ran with
  `timeout 31 stress -p 2 -timeout 30s <session-binary> -test.run <suite/TestAdd>`.
  Controller completed 30 iterations and model 20, zero reported failures
  before intentional outer timeouts. Binaries were removed.
- `make pre-check` passed but skipped static analysis.
  `RUN_SUBTEST=run_go make static-analysis STATIC_ANALYSIS_JOB=test_static_analysis_go`
  passed, as did `gofmt`, prescribed `gci` and `git diff --check`.
  Other static-analysis jobs and full repository tests were not run.

Tests were not rerun for the comment-only follow-up; formatting/diff checks
passed. It describes immutable registration rather than naming a backend.
Validation used stock dqlite fixtures; no PoC reconciliation was performed.

Lease-pin publication evidence is in the
[lease branch record](../4.1-lease-pin-conflict/status.md).

## U16 provenance snapshot (2026-11-11 migration)

[PR #23728](https://github.com/juju/juju/pull/23728), branch `4.1-agentbinary-conflict`, merged 2026-11-06 as `32be283d02` at head `2618d146da` with shared registration state. Both paths use `ON CONFLICT (version, architecture_id) DO NOTHING` and compare the retained registration, preserving `AlreadyExists` / `AgentBinaryImmutable`, idempotency and transaction usability. Earlier commits were cherry-picked onto `pg-poc/05-queries` 2026-11-05 (D43, identical patch IDs): `323807680f`→`fc66010f1a`, `6a66e5ed1b`→`c34b912abb`. These predate the shared-state refactor; reconcile the merged result. Default-backend tests pass on the recorded PoC stack; PostgreSQL and bootstrap not yet re-run.
