# 4.1-dbrepl-latest-row

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-11-04; merge `f7b596bf5c`

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-dbrepl-latest-row`, [#23730](https://github.com/juju/juju/pull/23730) | X16 | Apply explicit latest-ID selection and row join-back to the separate dbrepl change-stream query, preserving grouping, bounds, ordering and displayed fields. Merged 2026-11-04.

### Upstream register snapshot

`4.1-dbrepl-latest-row` | X16 | [#23730](https://github.com/juju/juju/pull/23730) | `49a1bdf808` | merged 2026-11-04; merge `f7b596bf5c`

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Dbrepl latest-row selection (X16)

Implemented locally on `4.1-dbrepl-latest-row`, worktree
`../4.1-dbrepl-latest-row`, independently based on upstream 4.1
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48`. Commit
`49a1bdf8085abbde324bedce53bbc189ed2efcfb` changes only `worker.go` and
`worker_test.go` in `internal/worker/dbrepl`. After explicit approval, the
branch was pushed and published as
[juju/juju#23730](https://github.com/juju/juju/pull/23730), from
`jameinel:4.1-dbrepl-latest-row` to `juju/juju:4.1`. The
[local PR text](../../archive/pr-descriptions/PR-4.1-dbrepl-latest-row.md) matches the published title/body.
The PR subsequently merged on 2026-11-04 as `f7b596bf5cb9`; the detailed
validation below records the original implementation, not a new test run.
No PoC integration was performed; dbrepl remains outside the PoC scope.

The assigned plan incorrectly described replication payload/progress.
Inspection showed that `changeStream` serves the interactive
`.change-stream` REPL command and prints rows; it has no replication state.
The user approved correcting that scope on 2026-10-30. Acceptance covers
the displayed header/fields and query meaning, not invented replication
behaviour. The separate changestream worker fix landed in juju/juju#23691;
this REPL query was not changed by it.

`MAX(c.id)` itself is standard SQL. The SQLite-specific behaviour was
selecting ungrouped edit type, namespace, key and timestamp from the row
supplying the maximum ID. The rewrite groups `change_log` by
`(namespace_id, changed)` to select maximum IDs, joins back to those rows,
then joins edit-type and namespace lookups. It preserves `id > 0`, latest
ID rather than latest timestamp, ascending ID ordering, all displayed
fields, coalescing and output formatting. Valid lookup references remain
enforced by the existing foreign keys; no schema/dependency change occurs.

Four regressions inspect the actual REPL output, normalising table
whitespace only. They cover multiple keys/namespaces, repeated keys whose
edit types/timestamps differ, timestamps reversed relative to IDs, empty
output, the first positive ID and excluded zero/negative-only keys. All
four pass with the original and rewritten SQL on both SQLite and actual
Dqlite. The final test helper was rechecked against the original SQL, then
the committed rewrite was restored; this is a portability improvement,
not a changed SQLite result contract.

### Validation on the local head

Commands ran from the new worktree, using its shared `_deps` symlink:

```sh
gofmt -w internal/worker/dbrepl/{worker.go,worker_test.go}
gci write --section standard --section default \
  --section 'Prefix(github.com/juju/juju)' \
  internal/worker/dbrepl/{worker.go,worker_test.go}
make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl \
  TEST_ARGS='-race -p 2 -count=1' \
  TEST_FILTER='TestDbReplSuite/TestChangeStream'
go test -race -p 2 ./internal/worker/dbrepl \
  -run 'TestDbReplSuite/TestChangeStream' -count=1
make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl \
  TEST_ARGS='-race -p 2 -count=1'
go test -race -p 2 ./internal/worker/dbrepl -count=1
make pre-check
golangci-lint run --concurrency 2 ./internal/worker/dbrepl/...
go build -p 2 ./internal/worker/dbrepl
git --no-pager diff --check
```

All passed. The Makefile supplies `dqlite,libsqlite3`, musl, CGO and linker
settings automatically. Final full package runs passed in 6.610s (Dqlite)
and 2.943s (SQLite). Initial targeted lint requested `strings.SplitSeq`
instead of `strings.Split` in the output helper; this was corrected and
lint rerun with zero issues. `make pre-check` skipped full static analysis.
The worktree was not an open IDE project; Go tests/build supplied build
validation.

Final actual-Dqlite stress used:

```sh
make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl \
  TEST_ARGS="-race -p 2 -c -o $binary"
timeout 31s stress -p 2 -timeout 30s "$binary" \
  -test.run TestDbReplSuite -test.timeout 30s
```

It completed 20 runs with zero failures before the intentional outer
exit 124. An earlier pre-lint-adjustment run completed 18 runs with zero
failures. The session-local test binary was removed. Full repository static
analysis/tests, full agent build, live interactive QA and PostgreSQL tests
were not run. Publication verification confirmed juju/juju#23730 open,
not a draft, targeting `4.1` at the recorded head; review/CI results have
not yet been assessed.

SQLite `EXPLAIN QUERY PLAN` on the existing changelog schema showed that
both versions scan the same qualifying source rows and use temporary
B-trees for grouping and ordering. The rewrite adds indexed fetches of
latest changelog rows, but performs edit-type/namespace joins only after
coalescing rather than for every source row. This is planner evidence,
not a benchmark or a guarantee of faster execution; when most keys have
only one row, the added fetches can add overhead. The command is an
interactive REPL path, not the production watcher.
