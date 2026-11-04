> Historical local draft recovered on 2026-11-04. The original metadata below
> predates publication: juju/juju#23731 already exists at `cfa8676090`.
> The common allocator / RETURNING changes described here are still
> uncommitted locally. Validation claims are historical, not newly verified.
> Any revised description or push requires fresh approval.

Destination: juju/juju, base 4.1, head jameinel:4.1-sequence-upsert-target
Action: proposed pull request; NOT published. Push and PR creation require approval.
Title: fix(sequence): target namespace conflicts when allocating sequence values

Both sequence allocators use an upsert without a conflict target, which
PostgreSQL does not accept for DO UPDATE. Their unqualified value expression
also conflicts with the value column exposed by the excluded row.

Target the sequence namespace primary key and qualify the existing stored
value in a private allocator shared by NextValue and NextNValues. Use
RETURNING to read the last allocated value from the upsert without a second
SELECT. NextValue returns the scalar directly; NextNValues constructs the
requested range. This keeps zero-based allocation, ordered contiguous
batches, zero-count no-ops, rollback and namespace isolation unchanged.
No schema or backend-specific code is introduced.

Add regressions for mixed allocations, static/prefix/empty namespaces,
zero-count no-ops after allocation, rollback of inserts and updates,
competing single/batch allocations, and continuation from imported sequence
values through export. These pass with both the original and updated SQL
on stock SQLite/Dqlite fixtures.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

From the branch worktree with the normal Juju dependencies:

1. Run the focused allocation and imported-sequence regressions:

   make run-go-tests TEST_PACKAGES=./domain/sequence/state \
     TEST_FILTER='"TestSequenceSuite|TestStateSuite/TestImportedSequencesContinue"' \
     TEST_ARGS='-p 2 -race'

2. Run all sequence packages:

   make run-go-tests TEST_PACKAGES='./domain/sequence/...' \
     TEST_ARGS='-p 2 -race'

Both commands use the standard dqlite,libsqlite3 build tags and passed.
The focused regressions also passed before the production SQL change.

Additional validation passed:

- make pre-check (its default skips static analysis).
- RUN_SUBTEST=run_go make static-analysis \
    STATIC_ANALYSIS_JOB=test_static_analysis_go
- make run-go-tests TEST_PACKAGES=./domain/sequence/state \
    TEST_ARGS='-p 2 -race -c -o <session-binary>'
- timeout 31 stress -p 2 -timeout 30s <session-binary> \
    -test.run 'TestSequenceSuite|TestStateSuite/TestImportedSequencesContinue'
  completed 10 runs with zero failures. The outer timeout ended the bounded
  run with status 124; two iterations were active at the final report.
  The temporary binary was removed.
- gofmt, prescribed gci ordering and git diff --check.

IDE build validation could not run because the dedicated worktree is not
an open IDE project; the standard test runner compiled the package instead.
No PostgreSQL execution, full repository tests or controller bootstrap/
migration QA has been performed.

## Documentation changes

No CLI/API workflow or public contract changes. Test comments describe the
allocation and persistence guarantees being exercised.

## Links

No upstream issue or Jira card is associated with this independently scoped
SQL portability change.
