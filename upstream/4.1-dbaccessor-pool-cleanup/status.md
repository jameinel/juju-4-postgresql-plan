# 4.1-dbaccessor-pool-cleanup

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

Fixed in PoC Stage 8 at `b6a7690d80` `fix(dbaccessor): close databases the tracker opens but does not keep`. Historical PoC validation: four new tests fail without the fix; race-enabled stress 214 runs, 0 failures. The tracker files at the original `pg-poc/08` base matched upstream `04bc64ef54`. [PR #23726](https://github.com/juju/juju/pull/23726) (`4.1-dbaccessor-pool-cleanup`, head `ee55bab5f4`) merged 2026-11-04 as `5259edff43`. Reconciled 2026-11-05 (default backend only): PoC `17cf0d3dff` dropped; `f77a1ffb9d` keeps the PoC `TrackedDBBackend`, `DBOpener` and `closeDB` on top of upstream's `currentDB`/close-replacement structure, with the ErrDBDead no-reopen check after the replacement close. PoC duplicates of upstream's close tests were dropped; `TestWorkerDoesNotReopenDeadDB` is kept. dbaccessor `-race` passes on SQLite.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-dbaccessor-pool-cleanup`, #23726 (U14) | Separate resource-ownership defect, not an SQL/schema concern. No D55-driven rework or follow-up indicated.

From `UPSTREAM-DONE.md`:

`4.1-dbaccessor-pool-cleanup`, [#23726](https://github.com/juju/juju/pull/23726) | U14 | Close unretained pools on setup and reopen failures while preserving caller/worker ownership of retained pools. Merged 2026-11-04.

From `UPSTREAM-FIXES.md`:

`4.1-dbaccessor-pool-cleanup` | U14 | [juju/juju#23726](https://github.com/juju/juju/pull/23726) | `ee55bab5f4` | merged 2026-11-04; merge `5259edff43`; PoC base since 2026-11-05; reconciled (default backend only)

### Upstream register snapshot

U14 | bug | `internal/worker/dbaccessor/tracker.go` | Leaked database pools. `openDatabase` did not close the pool when setting the foreign keys pragma failed; `newTrackedDBWorker` did not close it when model initialisation failed; `ensureDBAliveAndOpenIfRequired` never closed replacement pools it opened during reopen attempts that then failed their ping. Each leak kept a pool open until the process exited. | Fixed in PoC Stage 8 at `b6a7690d80` `fix(dbaccessor): close databases the tracker opens but does not keep`. Historical PoC validation: four new tests fail without the fix; race-enabled stress 214 runs, 0 failures. The tracker files at the original `pg-poc/08` base matched upstream `04bc64ef54`. [PR #23726](https://github.com/juju/juju/pull/23726) (`4.1-dbaccessor-pool-cleanup`, head `ee55bab5f4`) merged 2026-11-04 as `5259edff43`. Reconciled 2026-11-05 (default backend only): PoC `17cf0d3dff` dropped; `f77a1ffb9d` keeps the PoC `TrackedDBBackend`, `DBOpener` and `closeDB` on top of upstream's `currentDB`/close-replacement structure, with the ErrDBDead no-reopen check after the replacement close. PoC duplicates of upstream's close tests were dropped; `TestWorkerDoesNotReopenDeadDB` is kept. dbaccessor `-race` passes on SQLite.

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Dbaccessor pool cleanup (U14)

Implemented independently from refreshed upstream 4.1 base
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` in worktree
`../4.1-dbaccessor-pool-cleanup`, sharing `../juju/_deps`. Commit
`ee55bab5f471b1f0e19344a0dcb94a5a181808b3` changes only `tracker.go` and
`tracker_test.go` in `internal/worker/dbaccessor`. After explicit approval,
the branch was pushed to `jameinel/juju` and published as
[juju/juju#23726](https://github.com/juju/juju/pull/23726), targeting `4.1`.
The [local PR text](../../archive/pr-descriptions/PR-4.1-dbaccessor-pool-cleanup.md) matches the published
title/body. No published comments or review threads were modified.

The tracker leaked pools after foreign-key pragma setup failure, model
initialisation failure and failed verification of replacement pools.
Cleanup closes failed startup pools and each failed replacement before
another reopen or return. The caller still owns the original retained
pool; successful replacements remain open to be adopted by the loop.
Operation errors are preserved and close errors are logged. Pool limits
remain three idle/open connections, unchanged from the 4.1 base; the older
limit of two visible in `main` is not part of this diff.

Nine regressions cover pragma failure, model metadata and migration
failures, initialised-model usability, intermediate/terminal replacement
failures, reopen failure, cancellation, caller-owned pools and successful
replacement transfer. Before production edits, six regressions failed
because leaked pools were still usable; the successful model case passed.
The caller-ownership and cancellation cases were added subsequently.

### Validation on the published head

Commands ran from the new worktree. Formatting used `gofmt` and:

```sh
gci write --section standard --section default \
  --section 'Prefix(github.com/juju/juju)' \
  internal/worker/dbaccessor/{tracker.go,tracker_test.go}
go test -race -p 2 ./internal/worker/dbaccessor -count=1
make run-go-tests TEST_PACKAGES=./internal/worker/dbaccessor \
  TEST_ARGS='-race -p 2 -count=1'
make pre-check
golangci-lint run --concurrency 2 ./internal/worker/dbaccessor/...
go build -p 2 ./internal/worker/dbaccessor
git --no-pager diff --check
```

All passed. `make run-go-tests` automatically enables `dqlite,libsqlite3`
and supplies the existing musl/CGO/library/linker environment; its full
package race run passed in 18.698s. `make pre-check` skipped full static
analysis; explicit targeted golangci-lint reported zero issues. The worktree
was not an open IDE project, so command-line compilation supplied build
validation. `make go-test-alias` is also available; no `go-test` target
exists.

SQLite-backed tracker stress used:

```sh
go test -race -p 2 -c -o "$binary" ./internal/worker/dbaccessor
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run TestTrackedDBWorkerSuite -test.timeout 20s
```

It completed 18 runs with zero failures. Actual Dqlite stress used the
existing static archives with host gcc:

```sh
export CGO_ENABLED=1
export CGO_CFLAGS="-I$PWD/_deps/dqlite-deps-4.0-amd64/include"
export CGO_LDFLAGS="-L$PWD/_deps/dqlite-deps-4.0-amd64 -Wl,-Bstatic -ldqlite -luv -llz4 -lsqlite3 -Wl,-Bdynamic -lm"
go test -race -p 2 -tags=dqlite,libsqlite3 -c -o "$binary" \
  ./internal/worker/dbaccessor
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run TestTrackedDBWorkerSuite -test.timeout 30s
```

It completed 10 runs with zero failures. Both stress commands ended with
intentional outer timeout exit 124; both session-local binaries were
removed. An initial host-gcc Dqlite attempt without static archive selection
and `-lm` failed linking; the corrected flags passed without installations.
The Makefile target avoids manually supplying this environment.

At the 2026-10-30 publication follow-up, GitHub reported the PR open and
mergeable, with review required and CI still in progress. No failed checks
were observed; completed checks included CLA, conventional commits, SQL,
arm64/macOS builds and migration. This is not a full-green CI result.
Repository-wide tests/static analysis, full agent build, external bootstrap
QA and PostgreSQL integration tests were not run for this upstream branch.

PoC Stage 8 originally implemented U14 as `b6a7690d80`. The independent
upstream implementation landed as `5259edff43` and was reconciled on
2026-11-05: duplicate cleanup/tests were dropped while PoC backend
adaptations were retained. Default-backend validation passed; PostgreSQL
validation remains outstanding. The register owns the exact residual
commits and reconciliation evidence.

## U14 provenance snapshot (2026-11-11 migration)

Fixed in PoC Stage 8 at `b6a7690d80` `fix(dbaccessor): close databases the tracker opens but does not keep`. Historical PoC validation: four new tests fail without the fix; race-enabled stress 214 runs, 0 failures. The tracker files at the original `pg-poc/08` base matched upstream `04bc64ef54`. [PR #23726](https://github.com/juju/juju/pull/23726) (`4.1-dbaccessor-pool-cleanup`, head `ee55bab5f4`) merged 2026-11-04 as `5259edff43`. Reconciled 2026-11-05 (default backend only): PoC `17cf0d3dff` dropped; `f77a1ffb9d` keeps the PoC `TrackedDBBackend`, `DBOpener` and `closeDB` on top of upstream's `currentDB`/close-replacement structure, with the ErrDBDead no-reopen check after the replacement close. PoC duplicates of upstream's close tests were dropped; `TestWorkerDoesNotReopenDeadDB` is kept. dbaccessor `-race` passes on SQLite.
