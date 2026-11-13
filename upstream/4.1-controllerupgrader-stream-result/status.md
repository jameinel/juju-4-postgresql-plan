# 4.1-controllerupgrader-stream-result

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

- Owner/session: `533e88e8-ff4b-44c8-82d0-d01fac7ff80b`; user authorized
  review follow-up by session `17daff25-b14a-4ac3-a5f7-bcf4c44ab416`.
- Branch: `4.1-controllerupgrader-stream-result`.
- Worktree: `.worktrees/4.1-controllerupgrader-stream-result` under the primary checkout.
- Base: `bd970f693c8e5060973016fe9059108570fa5bd3`.
- GitHub rechecked 2026-11-13 20:51 UTC: the PR remains open at
  `23034abf0e303a197fd519e90327a1f66fc77d35`, with `REVIEW_REQUIRED`
  and aggregate checks `FAILURE`. No approval was present in the latest
  reviews. Triage current-head checks and complete review and functional
  QA; this check does not establish a common cause for CI failures.
- 2026-11-12: [#23806](https://github.com/juju/juju/pull/23806) is open at
  published head `23034abf0e303a197fd519e90327a1f66fc77d35`; human review
  required. New-head checks are still running, and the LXD charms smoke job
  failed during setup because Snap Store returned HTTP 408 while installing
  `core26`, before tests ran:
  https://github.com/juju/juju/actions/runs/43349540295/job/131637036334.
  Jenkins and Read the Docs passed on the preceding head
  `a380152f9beabb42bd64cfeb2dfff252439a6b9f`.
- With user approval, follow-up `23034abf0e303a197fd519e90327a1f66fc77d35`
  was pushed to `jameinel/juju:4.1-controllerupgrader-stream-result`.
  Simon's unresolved thread prompted removal of the controller lookup's
  misleading stream parameter and a stream-free result type. Shared helpers
  retain the model's same-transaction stream check. User approved preserving
  existing selection behavior and the helper; true stream filtering is not
  implemented.
- Scope finding: controller storage has no per-binary stream provenance.
  Stream-specific upgrade selection may use any controller-store binary;
  the model store instead gates on its current target stream. A real
  stream-membership design remains a separate decision and is not assigned
  to another branch.
- On the local head, `go test -race -p 2 -count=1 -timeout 120s
  ./domain/controllerupgrader/state ./domain/controllerupgrader/service`
  passed using SQLite; `make run-go-tests` for the state package passed
  with `-race -p 2 -count=1 -timeout 120s` and real dqlite. Scoped
  `golangci-lint`, compiled `txncheck`, `gci` and `git diff --check` passed.
  `make pre-check` skipped static analysis; PostgreSQL and live controller
  QA were not run.
- The user-approved `pr.md` was published as the PR body and verified.
  The approved Van Cleef response was published and verified at
  https://github.com/juju/juju/pull/23806#issuecomment-6841701600.
  The approved Simon reply was published and verified at
  https://github.com/juju/juju/pull/23806#discussion_r4558889339.
- Next: human review, current-head CI including the failed LXD setup, functional
  QA and eventual PoC reconciliation remain outstanding; do not resolve
  the thread without approval.

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
