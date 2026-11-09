# fix(database): close unretained dbaccessor pools

Failed foreign-key pragma setup, model initialisation and replacement-pool
verification leaked opened database pools. Repeated reopen attempts could
abandon additional pools.

Close unretained pools while preserving the original operation errors and
ownership of retained and successfully transferred pools. Nine regressions
cover startup failures, failed replacements, reopen failure, cancellation
and successful ownership transfer.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

make run-go-tests TEST_PACKAGES=./internal/worker/dbaccessor \
  TEST_ARGS='-race -p 2 -count=1'

Passed with actual Dqlite. SQLite-backed package race tests also passed.
Tracker stress: 18 SQLite-backed and 10 Dqlite runs, zero failures.
Targeted golangci-lint and package build passed. Bootstrap QA and full
repository static analysis were not run.

## Documentation changes

Documented pool ownership on the worker constructor. No CLI/API changes.

## Links

No linked issue or Jira card.
