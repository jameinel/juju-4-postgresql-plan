# 4.1-schema-layout

[Shared direction](../../AGENTS.md) |
[Upstream dashboard](../../UPSTREAM-PLAN.md) |
[Integration plan](../4.1-postgresql-tests/integration-plan.md)

## Current state

On 2026-11-20, rebuilt the layout branch as one commit on the published
ordering head `b622931495f0fb3a2242418c82e2fcb152183354`:
`9c8f98cd16289cb0e17b9be9ac587d59a683b285`. The dedicated
`../4.1-schema-layout` worktree is clean. The previous local tip
`4d92bd9501ea8ee00b799eda6761b01614aef630` is retained as
`4.1-schema-layout-before-order`. With explicit approval, pushed the new
head to `jameinel/juju:4.1-schema-layout` with a lease against
`9da8100e161f59ce5a86485e0725db1b8101154d` and verified the remote head.
No PR exists; PR creation requires separate approval.

The [PR draft](pr.md) targets upstream `4.1` while
[schema dependency ordering](../4.1-schema-dependency-order/status.md)
remains open, so reviewers can compare the combined ordering and layout
diff with the primary ordering-only proposal. It is not a request to merge
both PRs as independent changes. Native GitHub stack registration cannot
represent these cross-fork heads.

At this tree all 104 moved assets are exact Git renames of the ordering
parent. Patch discovery sorts unchanged filenames and hashes patch contents,
not paths. `go generate ./domain/schema/...` and `make rebuild-export`
produced no generated changes. SQLFluff 3.0.7 passed the CI glob; `gci`,
scoped `golangci-lint`, repository `txncheck` and `git diff --check` passed.

Race-enabled SQLite and real dqlite suites passed for
`./domain/schema/... ./core/database/... ./domain/agentbinary/...
./domain/modelmigration/... ./generate/ddlgen ./domain/sequence/state
./domain/controllernode/state`. SQLite used `go test -race -p 2 -count=1
-timeout 120s`; dqlite used `make run-go-tests` with the same packages and
test arguments. Race-compiled `domain/sequence/state` tests passed ten
bounded stress runs on each backend. The cumulative-upgrade tests skipped
because no post-patches are registered; full pre-check, live bootstrap and
PostgreSQL execution were not run.

### Previous independent layout state, 2026-11-19

On 2026-11-19 the user authorized takeover by session
`da709086-878a-463e-988a-dd325a2f386f` and chose the existing `dqlite`
directory name. The dedicated `../4.1-schema-layout` worktree and tracking
branch were restored from the existing remote head. The clean local head is
`4d92bd9501ea8ee00b799eda6761b01614aef630`; the remote still points to
`9da8100e161f59ce5a86485e0725db1b8101154d`. No PR exists. The
[PR draft](pr.md) is prepared for review; push and PR creation require
separate approvals.

The follow-up fixes the SQL CI glob: the prior command returned exit 2
because it did not match the relocated directories. SQLFluff 3.0.7 with
the updated glob passed all 31 controller and 38 model SQL files. The
changed upstream agent-version SQL also passed SQLFluff separately.
`git diff --check` passed.

The current upstream `4.1` head was
`59dcd6d951814fa098a11ca3be103314bb6894e8`, 111 commits beyond the
layout branch's base. A read-only Git merge-tree assessment of that head
with `4d92bd9501` was clean, including the upstream modification to
`model/sql/0027-agent-version.sql`. The merged tree has not been built or
tested; earlier Go validation below applies only to `9da8100e16`. Refresh
the upstream and PR state before publication, and verify CI on the actual
merge result. No rebase or new push was performed.

The independent [schema dependency-order PR](../4.1-schema-dependency-order/status.md)
also renumbers and rearranges these files. Its draft references the existing
layout comparison, but neither PR incorporates the other. Reconcile the
two changes at integration. The PostgreSQL test worktree remains untouched.

### Previous state, 2026-11-12

Owner/session: `0ec926e6-3858-4f43-bc9c-658024127e88`, assigned by the user
on 2026-11-12. Local branch `4.1-schema-layout` and worktree
`../4.1-schema-layout` were created from refreshed upstream `4.1`, base
`b36eefc4b34badcdef972c859e96ccb974c592e2`.

Local commit: `9da8100e161f59ce5a86485e0725db1b8101154d`
(`refactor(schema): move dqlite ddl under domain/schema/dqlite`).
The worktree is clean. The existing dirty harness work was preserved and
fast-forwarded onto this commit without a stash, reset or history rewrite.

The user explicitly approved the four exact branch-head pushes on 2026-11-12.
This head was pushed atomically with the other three to `jameinel/juju` and
verified with `git ls-remote`. No PR creation or further publication is
authorized.

## Approved scope and boundaries

Move `domain/schema/controller` and `domain/schema/model` under
`domain/schema/dqlite`. Update live imports, embeds, generation destinations,
schema-file discovery, release-DDL output paths, CI filters and references.
Preserve schema contents, patch ordering, hashes and public APIs.

Schema rigor is a first-class architectural concern. The PostgreSQL schema
will sit beside dqlite under `domain/schema/postgresql`, not inside a testing
directory. PostgreSQL runtime support is still outside this work.

This branch contains only the mechanical relocation. The SQL conversion
generator, PostgreSQL output, manual translation overrides, driver and
environment selection belong on the subsequent infrastructure branch.
Use PoC commit `11c1c182dd` as the source for the move-only patch, not the
PoC branch as the base.

## Acceptance evidence (2026-11-12)

- All 100 relocated files match their upstream Git blobs byte for byte,
  before and after `go generate ./domain/schema/...`. The final commit has
  100 exact renames plus path/documentation wiring (114 files, +61/-57).
- No live tracked reference to the old asset paths remains. Historical
  generated `.ddl` comments were excluded deliberately and kept unchanged.
- Race-enabled SQLite and real dqlite runs passed for:
  `./domain/schema/... ./core/database/... ./domain/agentbinary/...`
  `./domain/modelmigration/... ./generate/ddlgen ./domain/sequence/state`
  `./domain/controllernode/state`.
  SQLite used `go test -race -p 2 -count=1 -timeout 120s`; dqlite used
  `make run-go-tests` with the same `TEST_PACKAGES` and `TEST_ARGS`.
- The schema suites exercised fresh/idempotent/current cumulative behavior.
  The selected state consumers exercised both model and controller databases.
  No new skips or assertion changes were added.
- Race-compiled `domain/sequence/state` binaries ran `TestStateSuite`
  through bounded stress on both backends: `-p 2 -count 10 -timeout 30s`
  under an outer 35-second timeout, ten runs each, zero failures.
- Scoped `golangci-lint` reported zero issues; repository `txncheck` passed.
  `gci` covered all changed non-generated Go wiring, `gofmt` and
  `git diff --check` passed. IDE build succeeded with limited diagnostics;
  CLI compilation is the meaningful build evidence for ignored worktrees.

Logs are in session `0ec926e6-3858-4f43-bc9c-658024127e88`'s `files/`
directory (`schema-layout-*.log`). No PostgreSQL execution was claimed.

## Next actions

1. Obtain separate approval for any PR creation; the local [PR draft](pr.md)
   contains proposed functional QA, not a claim that bootstrap was run.
2. After ordering and layout land, move infrastructure-only commits onto
   the updated stock upstream base.
