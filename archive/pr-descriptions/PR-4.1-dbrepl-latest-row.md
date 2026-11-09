# fix(database): select latest change-stream rows explicitly

The `.change-stream` database REPL command selected `MAX(c.id)` alongside
ungrouped edit type, namespace, key and timestamp columns. SQLite takes
these columns from the row supplying the maximum ID, but other SQL
databases reject the ungrouped columns. `MAX(c.id)` itself is standard SQL.

Select the maximum change ID per `(namespace_id, changed)` first, then join
back to that row and its edit-type and namespace lookups. This preserves
the existing `id > 0` filter, per-namespace/key coalescing, ascending ID
order, displayed fields and output formatting. The latest row is selected
by ID, not timestamp. No replication state or broader backend support is
introduced.

Four regressions check the emitted header and row fields for repeated keys
with different edit types and reversed timestamps, multiple keys and
namespaces, empty results, the first positive ID, and excluded zero and
negative IDs. They pass with both the original and rewritten queries on
SQLite and actual Dqlite.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

Run the package race tests with actual Dqlite, then SQLite:

```sh
make run-go-tests TEST_PACKAGES=./internal/worker/dbrepl \
  TEST_ARGS='-race -p 2 -count=1'
go test -race -p 2 ./internal/worker/dbrepl -count=1
```

Run focused static analysis and compilation:

```sh
golangci-lint run --concurrency 2 ./internal/worker/dbrepl/...
go build -p 2 ./internal/worker/dbrepl
```

All commands passed. Actual-Dqlite suite stress completed 20 runs with zero
failures before the intentional outer timeout. `make pre-check` passed but
skipped full static analysis; targeted lint was run explicitly. Live REPL
QA, repository-wide tests/static analysis, full agent builds and PostgreSQL
tests were not run.

## Documentation changes

The query comment explains why latest-row selection is explicit. No CLI
or API workflow changes.

## Links

Related: juju/juju#23691 fixed the separate changestream worker query; this
change addresses the database REPL query independently.

**Issue:** none

**Jira card:** none
