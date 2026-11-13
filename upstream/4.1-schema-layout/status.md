# 4.1-schema-layout

[Shared direction](../../AGENTS.md) |
[Upstream dashboard](../../UPSTREAM-PLAN.md) |
[Integration plan](../4.1-postgresql-tests/integration-plan.md)

## Current state

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

1. Review the local [PR draft](pr.md); its functional controller QA is
   reproducible guidance, not a claim that bootstrap was run here.
2. Obtain explicit approval before opening a PR or publishing further changes.
3. After layout lands, move infrastructure-only commits onto the updated
   stock upstream base and retire this prerequisite branch when appropriate.
