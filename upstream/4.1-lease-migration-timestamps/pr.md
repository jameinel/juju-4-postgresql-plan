Lease migration test fixtures use SQLite datetime modifiers to calculate
start and expiry values. Compute UTC whole-second timestamps from one clock
reading on the test database's host and bind them as parameters instead.
This preserves the existing held, expired, model-isolation and lease-type
coverage without relying on SQLite-specific fixture expressions.

This changes tests only; production lease storage and migration behavior
are unchanged.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [x] ~Integration tests, with comments saying what you're testing~
- [x] ~doc.go added or updated in changed packages~

## QA steps

Not applicable: this changes test fixtures only and has no user-visible
behavior to exercise in a deployed controller.

## Documentation changes

None; the change is limited to test fixtures.

## Links

Follow-up to juju/juju#23686.
