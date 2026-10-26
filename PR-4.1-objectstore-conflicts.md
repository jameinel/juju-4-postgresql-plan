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
`UNIQUE constraint failed` error. It now fails with a plain "metadata uuid
already exists with different hashes or size" error, so callers can't
mistake it for content that is already stored. The existing metadata is
unchanged either way. The service generates a new `uuid` for every put, so
only a caller bug can hit this.

Tests cover matching and mismatched hashes and sizes, a path reused with a
different hash, and a reused `uuid` with the same or different hashes, with
and without a controller ID hint.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/objectstore/..." TEST_ARGS="-count=1 -p 4"
```

2. Restore the two `ON CONFLICT` clauses in `domain/objectstore/state/state.go`
   and rerun. Only `TestPutMetadataWithSameUUIDDifferentHashes` and
   `TestPutMetadataWithControllerIDHintSameUUIDDifferentHashes` fail, with
   `UNIQUE constraint failed: object_store_metadata.uuid`.

3. Deploy the same charm twice, which reuses the stored charm by hash. Both
   applications become active.

```sh
$ juju bootstrap lxd qa-objectstore
$ juju add-model m1
$ juju deploy juju-qa-test a
$ juju deploy juju-qa-test b
$ juju status
```

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
