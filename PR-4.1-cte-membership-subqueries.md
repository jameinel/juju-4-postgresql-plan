SQLite accepts `IN cte` and `NOT IN cte` as shorthand for membership in a
single-column CTE. Explicit membership subqueries express the same operation
without relying on that SQLite-specific syntax.

Replace eleven expressions with `IN (SELECT column FROM cte)` or its `NOT IN`
equivalent, retaining the existing CTEs and selecting their intended columns.
This preserves endpoint refresh filtering, scoped address and constraint
cleanup, storage lifecycle guards, operation completion and size accounting,
and departed-unit membership. No schema, API or transaction changes.

Add a departure-scoping regression covering empty, partial and full membership,
units scoped to another relation, and units belonging to another application.
It passes with both the original and rewritten SQL on SQLite/Dqlite.

This independently targets `4.1`. The operation-size review #23693 also touches
`prune.go`, but owns different expressions and is not a prerequisite.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

Run the affected state-package race suites with stock SQLite/Dqlite fixtures:

```sh
make run-go-tests \
  TEST_PACKAGES='./domain/application/state ./domain/removal/state/model ./domain/operation/state ./domain/relation/state' \
  TEST_ARGS='-p 2 -race'
```

All four packages passed. Focused existing regressions passed before and after
rewriting the SQL; the new departure-scope regression also passed separately
with the original SQL.

`make pre-check`, Go static analysis, `gofmt`, `gci` and `git diff --check`
passed. The race-built departure-scope regression completed 38 bounded stress
iterations with zero reported failures; the outer timeout stopped the run
with two iterations active. The test binary was removed.

PostgreSQL execution, the full repository suite and live deployment/migration
QA were not run. The change makes these membership expressions portable; it
does not claim the affected packages are otherwise PostgreSQL-compatible.

## Documentation changes

None. No user-facing workflow changes.

## Links

Related independent operation-size review: #23693.
