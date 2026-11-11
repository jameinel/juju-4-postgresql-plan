# refactor(operation): use portable boolean and size expressions

Two queries in `domain/operation/state` rely on SQLite converting values
between types. This rewrites them in standard SQL. Results are unchanged.

**Filter flags in `GetOperations`.** The query binds Go `bool` flags but
compared them with `0` and `1`. It now compares them with `FALSE` and `TRUE`.

**Size estimate used for pruning.** `computeTableSize` applied `octet_length`
to integer, boolean and timestamp columns, but `octet_length` is only defined
for strings. Those columns are now measured as
`octet_length(CAST(column AS TEXT))`, which is what SQLite already did
implicitly. Each caller marks its columns with `textColumn` or
`nonTextColumn`.

New tests cover every combination of the five filter flags, the exact number
of bytes measured for each column type, and the pruning decision around the
size limit. They pass against both the old and the new queries.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/operation/..." TEST_ARGS="-count=1 -p 4"
```

2. Check that the new tests catch a change in behaviour:

   - In `domain/operation/state/query.go`, change
     `$queryFlags.has_units = TRUE` to `= FALSE`.
     `TestGetOperationsFilterFlagCombinations` fails.
   - In `domain/operation/state/prune.go`, change `CAST(%q AS TEXT)` to
     `CAST(%q AS INTEGER)`. `TestComputeTableSizeByColumnType`,
     `TestEstimateOperationSizeExactBytes` and `TestPruneBySizeAtThreshold`
     fail.

3. On a controller, run some operations and filter them. Each listing shows
   only the matching operations.

```sh
$ juju bootstrap lxd qa-operations
$ juju add-model m1
$ juju deploy ubuntu
$ juju wait-for application ubuntu
$ juju exec --unit ubuntu/0 -- echo one
$ juju exec --application ubuntu -- echo two
$ juju exec --machine 0 -- echo three
$ juju operations --units ubuntu/0
$ juju operations --apps ubuntu --status completed
$ juju operations --machines 0
```

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
