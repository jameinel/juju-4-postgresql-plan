# fix(objectstore): use a single conflict clause when putting metadata

`putMetadata` in `domain/objectstore/state` inserted metadata with two
conflict clauses, `ON CONFLICT (sha_256) DO NOTHING` and
`ON CONFLICT (sha_384) DO NOTHING`. Repeating `ON CONFLICT` is a SQLite
extension. This replaces them with a single `ON CONFLICT DO NOTHING`.

The table's only unique constraints are the `uuid` primary key and the two
hash indexes, so hash conflicts behave as before: the existing row with the
same hash and size is reused, and a size mismatch returns
`ErrHashAndSizeAlreadyExists`.

A reused `uuid` with different hashes or size used to fail with a raw
`UNIQUE constraint failed` error. It returns the documented
`objectstoreerrors.ErrMetadataUUIDAlreadyExists` sentinel, so callers can
match it with `errors.Is` without mistaking it for content that is already
stored. The existing metadata is unchanged. The service generates a new
`uuid` for every put, so only a caller bug can hit this. The sentinel keeps
the error text from the initial version of this PR.

Tests cover matching and mismatched hashes and sizes, a path reused with a
different hash, and a reused `uuid` with matching metadata, different hashes
or only a different size, with and without a controller ID hint. Service
tests verify that both put methods preserve the wrapped UUID-conflict
sentinel and return no UUID on failure.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/objectstore/..." \
  TEST_ARGS="-race -count=1 -p 2"
```

2. Verify the UUID-reuse cases match `ErrMetadataUUIDAlreadyExists`, not
   `ErrHashAndSizeAlreadyExists`, for different hashes and for a size-only
   mismatch. Existing metadata and paths remain unchanged, and the rejected
   controller hint is not recorded. Both service put methods preserve the
   sentinel through error wrapping.

3. Run focused static analysis.

```sh
make pre-check
golangci-lint run ./domain/objectstore/...
```

4. Deploy the same charm twice, which reuses the stored charm by hash. Both
   applications become active.

```sh
$ juju bootstrap lxd qa-objectstore
$ juju add-model m1
$ juju deploy juju-qa-test a
$ juju deploy juju-qa-test b
$ juju status
```

## Documentation changes

The domain errors package documents `ErrMetadataUUIDAlreadyExists` beside
`ErrHashAndSizeAlreadyExists`. No CLI or API workflow changes.

## Links

**Issue:** none

**Jira card:** none
