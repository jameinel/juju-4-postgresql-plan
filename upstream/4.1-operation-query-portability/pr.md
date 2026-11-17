# refactor(operation): use portable boolean and size expressions

Make operation filtering and pruning-size estimation independent of SQLite's
implicit type conversions.

`GetOperations` compares its Go boolean flags with `TRUE` and `FALSE`; the
filtering behavior is unchanged.

Replace dynamically constructed table-size and count queries with static
aggregates. The pruning estimate measures variable-length text, uses fixed
per-row estimates for UUIDs and non-text fields, and includes referenced
object-store content. Nullable text contributes zero without hiding the other
fields in its row. The object-store metadata size uses an explicit text cast
when measuring its textual representation.

The estimate is not the database's stored byte count. It can change when
pruning happens, particularly near the configured size limit. The existing
average-based deletion count, rounding and deletion order are unchanged, so
the limit is not a hard cap. Constant row allowances document the fields they
cover; tests compare the estimate with representative content and flag
changes to the relevant table columns or types for review.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

Use a controller built from this branch and an LXD cloud. Create a model and
deploy a machine charm:

```sh
make install
juju bootstrap lxd qa-operations --build-agent
juju add-model qa-operations
juju model-config max-action-results-age=0 max-action-results-size=100M
juju deploy ubuntu
juju wait-for application ubuntu
```

Create a large completed operation and a smaller one, then check filtering.
Both operations should appear in the unit and application listings. The
machine listing should not include these unit-targeted operations.

```sh
juju exec --unit ubuntu/0 -- 'head -c 3145728 /dev/zero | base64' > /dev/null
juju exec --unit ubuntu/0 -- echo small
juju operations --units ubuntu/0
juju operations --apps ubuntu --status completed
juju operations --machines 0
```

Lower the size limit to trigger pruning and list the operations again. The
older large operation should disappear while the newer one remains. Allow
time for the controller to process the model configuration change. Because
pruning is approximate, do not expect an exact byte threshold.

```sh
juju model-config max-action-results-size=1M
juju operations --units ubuntu/0
```

Remove the controller after the check:

```sh
juju destroy-controller qa-operations --destroy-all-models --destroy-storage
```

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
