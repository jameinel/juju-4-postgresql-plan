# refactor(model): drop iif from the model connection info query

`GetModelConnectionInfo` in `domain/model/state/controller` derived
`has_import_claim` with `IIF(mmi.model_uuid IS NOT NULL, TRUE, FALSE)`.
`IIF` is SQLite-specific, and the wrapper is redundant: `IS NOT NULL`
already yields the boolean. This selects `mmi.model_uuid IS NOT NULL`
directly, as other state queries already do (for example, the relation
unit status query in `domain/relation/state`).

The result is unchanged. `model_migration_import.model_uuid` has a unique
index, so the left join yields at most one row per model, and the
expression is never NULL. The query's other columns, its joins and its
not-found handling are unchanged.

This is a portability and readability improvement, not a fix for a
current Dqlite defect. The existing tests already cover each combination
of activated and not-activated models, with and without an import claim,
plus a missing model. They pass with both the old and new SQL, so no tests
were added.

The `IIF` expressions in the controller schema views (`0007-model.sql`)
are DDL and are deliberately left for a separate change.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- ~[ ] Comments saying why design decisions were made~
- ~[ ] Go unit tests, with comments saying what you're testing~
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests \
  TEST_PACKAGES="./domain/model/state/... ./domain/model/service/..." \
  TEST_ARGS="-race"
```

2. Run focused static analysis.

```sh
golangci-lint run ./domain/model/...
```

## Documentation changes

None.

## Links

None.
