# fix(lease): resolve competing pins with ON CONFLICT

`PinLease` in `domain/lease/state` inserted a `lease_pin` row and, inside
the same transaction, treated a unique-constraint error as success. SQLite
keeps a transaction usable after a failed statement, but standard SQL
engines such as PostgreSQL abort it, so relying on a caught constraint
error is not portable.

This resolves repeated and competing pins in the statement itself with
`ON CONFLICT (lease_uuid, entity_id) DO NOTHING`, targeting the existing
`idx_lease_pin_lease_entity` unique index. The result is unchanged:
pinning an already pinned lease for the same entity succeeds without
adding a row, and pinning a lease that does not exist is a no-op. Any
other constraint failure is now returned rather than ignored.

This is a portability improvement, not a fix for a current Dqlite defect.
The new tests pass with both the old and new SQL on Dqlite.

Tests cover a repeated pin keeping the original row, continued pin and
unpin use after the conflict, concurrent pins of one lease by two
entities, and pinning a missing lease.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/lease/..." \
  TEST_ARGS="-race -count=3"
make run-go-tests TEST_PACKAGES="./domain/lease/state/..." \
  TEST_ARGS="-race -count=20 -run TestStateSuite/TestPin"
```

2. Run focused static analysis.

```sh
golangci-lint run ./domain/lease/...
```

## Documentation changes

None.

## Links

None.
