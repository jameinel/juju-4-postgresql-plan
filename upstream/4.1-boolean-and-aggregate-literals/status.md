# 4.1-boolean-and-aggregate-literals

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Claimed 2026-11-11 by session `24713b22-6d3f-4622-9c56-7756485516d1`
with coordinator's exclusive reservation window. The coordinator authorised
takeover by session `9c43947c-a0e6-40c2-a3dc-af93e0bbe9a8` on 2026-11-12.

- Branch: `4.1-boolean-and-aggregate-literals`; worktree:
  `.worktrees/4.1-boolean-and-aggregate-literals`, with shared `_deps`.
- Verified upstream 4.1 base: `bd970f693c8e5060973016fe9059108570fa5bd3`.
- Head: `039998905e18a9d414e2a36e13128eb0aaad7a02` (2026-11-12),
  pushed to `jameinel/juju` and verified as the PR and remote branch head.
- PR: [juju/juju#23809](https://github.com/juju/juju/pull/23809), open;
  published [description](pr.md) and replies verified 2026-11-12.
- GitHub rechecked 2026-11-13 20:51 UTC: the PR remains open at that head.
  Simon approved on 2026-11-13, but GitHub still reports `REVIEW_REQUIRED`
  and aggregate checks `FAILURE`. The previous head's CI static analysis
  failed on govulncheck `GO-2026-6859`; whether it predates this branch is
  unverified. Next: triage current-head failures, complete the remaining
  review gate and functional QA, and investigate the finding if it recurs.
  Kubernetes functional QA was omitted at the coordinator's direction.

2026-11-12 review disposition: Simon Richardson requested clarification or
replacement of `COUNT`; with coordinator agreement the follow-up selects the
nullable instance ID directly, treating no row as invalid credentials. Van
Cleef suggested including the adjacent `IFNULL` cleanup and describing the
subordinate fixture fix; the coordinator approved both. This branch changes
that one occurrence to `COALESCE`, superseding the earlier split decision
below; the remaining occurrences stay outside this branch. The separate
breakout must omit this occurrence or reconcile overlap. Published replies:
[Simon](https://github.com/juju/juju/pull/23809#discussion_r4558803973)
and [Van Cleef](https://github.com/juju/juju/pull/23809#issuecomment-6841552781).
Local copies: [Simon](reply-simonrichardson.md) and
[Van Cleef](reply-lee-van-cleef.md). Simon's thread was not resolved.

At `039998905e`, focused SQLite regressions and both complete affected
package suites passed with `-race -p 2 -count=1`. Both complete packages
passed real dqlite via `make run-go-tests` with
`TEST_ARGS='-race -p 2 -count=1 -timeout 720s'` and `TEST_FILTER=.`.
Scoped `golangci-lint`, repository-wide `txncheck`, `gci` and
`git diff --check` passed. `STATIC_ANALYSIS=1 make pre-check` completed
successfully but skipped `govulncheck`, `go-licenses` and `shfmt` because
they were unavailable. CI's vulnerability failure is not resolved by this
local result. PostgreSQL and functional QA were not run.

2026-11-11: The existing subordinate-exclusion test attached its second unit
to the principal application and charm, so it did not exercise a subordinate.
Correcting that fixture is part of validating the changed predicate.

2026-11-11: Coordinator directed that the separate `IFNULL` cleanup remain
outside this branch. X7's published juju/juju#23751 covers `IIF`, not this
`IFNULL`; the latter remains a candidate. PoC commit `8930466847` changes
the join two lines above U20's predicate. Applying its patch to this worktree
with `git apply --check` fails because the modified predicate is patch context.
Whichever change lands second must resolve the overlap and retain both edits.

The coordinator subsequently chose to split `IFNULL` cleanup into a separate
breakout. A case-insensitive repository-wide search found five production
calls (two network, one access, one relation watcher, one change stream)
and nine test calls (two migration, one resource, six removal). This inventory
supersedes the incomplete seven-call inventory previously presented. The
separate breakout is a coordinator handoff, not assigned here.

2026-11-11: At `5f8af01651`, SQLite package suites passed with
`go test -race -p 2 -count=1 -timeout 300s` for `./domain/network/state`
and with the same options and a 180s timeout for
`./domain/agentpassword/state`. Real dqlite package suites passed with
`make run-go-tests`, `TEST_ARGS='-race -p 2 -count=1 -timeout 720s'` and
`TEST_FILTER=.` for both packages. Scoped `golangci-lint` and repository-wide
`txncheck` passed. `gci` and `git diff --check` passed. `make pre-check` ran
but skipped full static analysis because `STATIC_ANALYSIS` was unset.
PostgreSQL and functional QA were not run. The first combined dqlite selector
was rejected by the Makefile's shell quoting, and the first SQLite password
suite failed when a new missing-instance fixture deleted a row with a
dependent status; the fixture was fixed and both complete suites subsequently
passed.

## Scope and acceptance

- Correct boolean comparisons in `domain/network/state` machine bindings and
  controller API address queries (the latter was renamed from
  `GetUnitAPIAddresses`); retain existing machine, Kubernetes, address-scope
  and subordinate-selection behavior.
- Correct the nonaggregated instance ID in the
  `domain/agentpassword/state` machine-password query by selecting the
  instance ID directly. The schema limits each machine to one cloud instance
  row. Preserve matching, missing, invalid and unprovisioned cases, including
  NULL and empty instance IDs.
- Change `IFNULL` in the modified network query to `COALESCE`; other
  occurrences remain for separate cleanup.
- No schema changes or PostgreSQL runtime dependency. Explain why the SQL
  changes remove the portability defects; validate affected behavior on
  SQLite and real dqlite, with focused regressions and affected package suites.
- Prerequisites: current upstream 4.1; no stack or dependency identified.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

### Upstream register snapshot

`4.1-boolean-and-aggregate-literals` | U20 | not proposed |  | PoC implementation on Stage 5 (`30e2bca25a`, `ada2a852ee`) and Stage 6 test SQL (`fa75871904`); scoped in UPSTREAM-PLAN.md

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

#### `4.1-boolean-and-aggregate-literals` — U20, portability

`domain/network/state` `GetMachineAppBindings` and `GetUnitAPIAddresses`
compare booleans with `0`/`1`; use `FALSE`/`TRUE`. `domain/agentpassword`
`MatchesMachinePasswordHashWithNonce` selects `mci.instance_id` beside
`COUNT` without grouping; use `MAX(mci.instance_id)` (a machine has at
most one cloud instance row). PoC `30e2bca25a`, `ada2a852ee`; the
agentpassword test SQL fix is Stage 6 `fa75871904`.

## U20 provenance snapshot (2026-11-11 migration)

Fixed, Stage 5, `30e2bca25a` (`FALSE` / `TRUE`) and `ada2a852ee` (`MAX(mci.instance_id)`); test SQL `fa75871904` (Stage 6; was `283a00f630` before the 2026-11-06 restack). Not proposed.
