Register agent binaries without overwriting an existing version and
architecture mapping in either controller or model state.

`ON CONFLICT (version, architecture_id) DO NOTHING` atomically inserts a new
registration or leaves the existing mapping untouched. When the insert
affects no rows, read the registered object UUID and compare it with the
supplied UUID to distinguish a repeated registration from an attempt to
replace an immutable binary.

Handle an expected duplicate as an insert outcome rather than a failed SQL
statement, keeping the transaction usable for the comparison. The explicit
conflict target handles only the registration key; other database errors
still propagate.

Registration semantics remain unchanged:

- New registrations succeed.
- Re-registering the same object returns `AlreadyExists`.
- Registering a different object for the same version and architecture returns
  `AgentBinaryImmutable` without replacing the original mapping.
- Missing objects and unsupported architectures retain their existing errors.

The implementation uses SQLite/dqlite's conflict handling and affected-row
results. It adds no backend-specific code or schema changes.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with descriptive names identifying the behaviour tested
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

Use the checkout's normal dqlite dependencies and Make test environment:

```sh
make run-go-tests \
  TEST_PACKAGES='./domain/agentbinary/state/controller ./domain/agentbinary/state/model' \
  TEST_FILTER='Test.*StateSuite/TestAdd' \
  TEST_ARGS='-p 2 -race'

make run-go-tests \
  TEST_PACKAGES='./domain/agentbinary/state/controller ./domain/agentbinary/state/model' \
  TEST_ARGS='-p 2 -race'

RUN_SUBTEST=run_go make static-analysis \
  STATIC_ANALYSIS_JOB=test_static_analysis_go
```

All three commands passed locally. `make pre-check` also passed, but its
default configuration skips static analysis; the explicit Go lint command
above ran successfully.

Existing tests cover new, repeated and different-object registrations,
missing objects and unsupported architectures. New tests in both state
suites check that both conflict cases leave the same transaction able to
perform another write. They also verify that the original mapping remains
unchanged and that the probe write rolls back with the returned error.

For bounded race/stress validation, each package was compiled separately
using `make run-go-tests TEST_ARGS='-p 2 -race -c -o <test-binary>'`, then run
with `timeout 31 stress -p 2 -timeout 30s <test-binary> -test.run <suite/TestAdd>`.
The controller completed 30 iterations and the model completed 20, with zero
reported failures before the outer timeouts. Test binaries were removed.

Validation used stock dqlite fixtures only. The full repository test suite,
other static-analysis jobs and bootstrap/migration QA were not run. No facade
or migration behaviour changes are included.

## Documentation changes

~Not applicable: no CLI, API or user-workflow changes.~

## Links

- [Branch comparison](https://github.com/juju/juju/compare/4.1...jameinel:juju:4.1-agentbinary-conflict)
- [Commit](https://github.com/jameinel/juju/commit/323807680f7a5249f65d2d7378c0c2158cb3edee)

**Issue:** ~Not linked.~

**Jira card:** ~Not linked.~
