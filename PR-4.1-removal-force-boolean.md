# fix(schema): declare removal force as boolean

`removal.force` was declared `INT NOT NULL DEFAULT 0`, but every writer binds
a Go `bool` and the only reader scans into one. The stored 0 and 1 values were
already right, but the generated 4.1 export type carried the flag as `int64`.

This declares the column `BOOLEAN NOT NULL DEFAULT FALSE` and regenerates the
4.1 export types, so `v4_1_0.Removal.Force` is a `bool`. A new `Removal` delta
in the 4.0.12 to 4.1.0 import transform converts 0 and 1 to `false` and `true`,
and rejects any other value. No production reader or writer changes.

4.1 is in beta, so the base schema file is edited in place. A model database
created by an earlier 4.1 beta fails the schema hash check. 4.0 models reach
4.1 by migration, so released versions are not affected.

This conflicts with #23680, which also adds a transform delta and extends the
same fixtures. Whichever lands second will be rebased and regenerated.

Tests cover the delta, the column declaration and stored values, the import
and export round trip, an end-to-end 4.0.12 import, and the recovery dump.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/modelimport/... ./domain/export/... ./domain/schema/... ./domain/recovery/... ./internal/recovery/..." TEST_ARGS="-count=1 -p 4"
make run-go-tests TEST_PACKAGES="./domain/removal/... ./domain/secret/state/... ./domain/unitstate/state/... ./apiserver/facades/controller/migrationtarget/..." TEST_ARGS="-count=1 -p 4"
```

2. Check the generated types are current. `make rebuild-export` leaves no
   diff.

3. Migrate a 4.0 model to 4.1 and check that it keeps working.

```sh
$ juju bootstrap lxd src40
$ juju add-model moveme
$ juju deploy juju-qa-test
$ juju bootstrap lxd dst41
$ juju migrate src40:moveme dst41
$ juju add-unit -m dst41:moveme juju-qa-test
$ juju debug-log -m dst41:moveme --replay --no-tail --level WARNING
```

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
