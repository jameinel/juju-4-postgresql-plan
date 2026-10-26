# refactor(changestream): select latest change row explicitly

The change stream coalesces several changes to the same namespace and
`changed` value into one event, which must describe the latest change. It did
this with:

```sql
SELECT MAX(c.id), c.edit_type_id, n.namespace, changed, created_at
  FROM change_log c ... WHERE c.id > ?
  GROUP BY c.namespace_id, c.changed ORDER BY c.id;
```

`edit_type_id` and `created_at` are bare columns next to an aggregate. SQLite
takes them from the row with the maximum id, but standard SQL rejects this or
leaves the row unspecified.

The query now selects `MAX(c.id)` per group in a common table expression and
joins back to `change_log` by primary key, so every column of an event comes
from the latest row. Ordering, coalescing and the high-water mark are
unchanged. `EXPLAIN QUERY PLAN` shows the same range scan plus one primary key
lookup per event, and the timing is the same.

New tests check that each event matches the latest row in every column, that
rows at or below the high-water mark are excluded, that pruning doesn't change
the events, and that concurrent writes are streamed correctly.

`internal/worker/dbrepl/worker.go` has a similar query that is not changed
here.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite and `-race`.

```sh
make run-go-tests TEST_PACKAGES="./internal/changestream/... ./domain/changestream/... ./internal/worker/changestream/..." TEST_ARGS="-count=1 -p 4 -race"
```

2. Run the new tests under `stress`, building with the cgo environment that
   `make go-test-alias` prints. 223 runs passed.

```sh
go test -c -race -tags=dqlite,libsqlite3 \
  -ldflags "-linkmode 'external' -extldflags '-static'" \
  -o /tmp/stream.test ./internal/changestream/stream/
cd internal/changestream/stream

timeout 130 stress -timeout 60s -p 4 /tmp/stream.test \
  -test.run '^TestStreamSuite$/^(TestConcurrentWritesAreStreamedAsLatestRows|TestLatestRowOfEachChangeAcrossTerms|TestReadChangesReturnsLatestRowOfEachChange|TestReadChangesOnlyReturnsRowsAboveUpperBound|TestReadChangesIgnoresPrunedRows)$'
```

3. Bootstrap, deploy and remove an application, and check the controller log
   has no `changestream` errors.

```sh
$ juju bootstrap lxd qa-changestream
$ juju add-model m1
$ juju deploy ubuntu
$ juju remove-application ubuntu
$ juju debug-log -m controller --replay --no-tail --level WARNING
```

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
