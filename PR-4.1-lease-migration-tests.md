**Title:** `test(lease): assert seed success in migration state tests`

**Branch:** `4.1-lease-migration-tests` (base `upstream/4.1` @ `04bc64ef54`)
**Commit:** `7e3e903650`
**Target:** `juju/juju` `4.1`

---

Test-only change to `domain/lease/state/migration_test.go`. No production
code is modified.

Before this change, the seeding transactions in the
`GetApplicationLeadershipForModel` tests discarded the error returned by
`StdTxn`. A failed insert left the table empty and the tests carried on:
`TestGetApplicationLeadershipForModelExpired` asserts `HasLen, 0`, so it
would pass vacuously. The fixtures also used date-only values
(`date('now')`), which cannot distinguish "started earlier today" from
"expired", and nothing covered a mix of held and expired leases or leases
from another model.

Changes:

- A single `insertLease` helper asserts the seed transaction succeeds.
- Fixtures use `datetime('now', <modifier>)` for `start` and `expiry`.
- Added `TestGetApplicationLeadershipForModelHeldAndExpired`: seeds a held
  lease, an expired lease and a held lease in a different model, and checks
  only the held lease for the requested model is returned.

Not changed: the query uses `expiry >= date('now')`, so expiry is compared
at date granularity. A lease that expired earlier today is still reported as
held. That is production behaviour and out of scope here; flagging it for
separate discussion.

## Checklist

- [x] Code style: imports ordered (`gci`), good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests~ (test-only change)
- [ ] ~doc.go added or updated in changed packages~

## QA steps

```sh
go test ./domain/lease/state/
```

Result: `ok  github.com/juju/juju/domain/lease/state`

Not run: `make pre-check`, race/stress (no goroutines or mutexes involved).

## Documentation changes

None.

## Links

**Issue:** n/a
**Jira card:** n/a
