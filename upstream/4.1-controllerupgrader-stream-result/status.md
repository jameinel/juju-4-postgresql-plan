# 4.1-controllerupgrader-stream-result

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

- Owner/session: `533e88e8-ff4b-44c8-82d0-d01fac7ff80b`.
- Branch: `4.1-controllerupgrader-stream-result`.
- Worktree: `.worktrees/4.1-controllerupgrader-stream-result` under the primary checkout.
- Base: `bd970f693c8e5060973016fe9059108570fa5bd3`.
- Recorded phase: published.

[#23806](https://github.com/juju/juju/pull/23806), head `a380152f9beabb42bd64cfeb2dfff252439a6b9f`, based on PoC `274e1a2e0211a56dfdcef6591c42cc75ce52b0d5`; human review required and CI pending. SQLite and real-dqlite state package race suites, scoped lint, txncheck, gci and diff checks passed; pre-check skipped static analysis. PostgreSQL and live controller QA not run. Complete QA, review and CI; reconcile PoC after landing.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### `4.1-controllerupgrader-stream-result` — U25, remove SQL pass-through

**Status:** PoC Stage 5 `274e1a2e02` implements the approved approach;
the independent upstream breakout is published as
[#23806](https://github.com/juju/juju/pull/23806), head `a380152f9b`.
Human review, CI and live QA remain.

`GetAllAgentStoreBinariesForStream` in
`domain/controllerupgrader/state/controller.go` and `model.go` currently
projects the caller's integer stream argument as a constant SQL result
column. The parameter has no type context, and it contributes no stored
information: every returned agent binary receives the same stream argument.
Both methods collect all rows with `GetAll` and return a Go slice; there is
no paging or cursor contract to preserve.

Select only the stored version and architecture fields, using an appropriate
result type, and set `AgentBinary.Stream` directly from the method argument
when constructing the result slice. Remove the unnecessary parameter from
query preparation and execution. Preserve the model's target-stream check,
empty results, version parsing errors and existing controller-store behavior;
do not add stream filtering, change storage or mask incorrect stored types.

Cover controller and model results with several binaries and more than one
supported stream, empty stores and a mismatched model stream. Validate SQLite
and real dqlite. PostgreSQL exposed the parameter-typing issue but its runtime
or harness is not part of this independent upstream patch. Record the branch,
source commits, validation and eventual PoC reconciliation separately.

## U25 provenance snapshot (2026-11-11 migration)

Fixed on PoC Stage 5 as `274e1a2e02`: select stored version/architecture fields and populate `AgentBinary.Stream` directly from the method argument. Focused PostgreSQL and real-dqlite checks passed at `09=85e4e6b122`; see Stage 5. Independent upstream [#23806](https://github.com/juju/juju/pull/23806), head `a380152f9b`, awaits review, CI and live QA; reconcile after landing.
