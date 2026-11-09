# refactor: use standard sql equality and join syntax

SQLite accepts some syntax that standard SQL does not. This replaces it in
four queries and two model triggers. Each statement returns or changes the
same rows as before. There are three commits, so each can be reviewed or
dropped on its own.

1. `==` becomes `=` in three queries: keymanager `DeletePublicKeysForUser`,
   storageprovisioning `GetVolumeAttachmentIDs` and crossmodelrelation
   `ValidateApplicationAndEndpointsForOffer`.
2. removal `GetDetachInfoForStorageAttachment` used
   `FROM storage_attachment sa, fulfilment f JOIN storage_instance si ON sa...`.
   In standard SQL `JOIN` binds tighter than the comma, so `sa` is out of
   scope in that `ON` clause. The comma is now `CROSS JOIN`. `fulfilment` is
   an aggregate with exactly one row, so the result is the same.
3. `==` becomes `=` in the machine-provisioned storage triggers in
   `domain/schema/modeltriggers.go`.

Commit 3 changes the text of an applied schema patch. A model database
created by a 4.1 beta fails the schema hash check after this change. 4.0
models reach 4.1 by migration, so released versions are not affected. If
that is a problem before 4.1 is released, commit 3 can be dropped.

New tests cover the cases the existing tests missed: the keymanager clean-up
of key records no model uses, detach info when no attachment is alive, and
the triggers not firing for a second attachment or for deleting an attachment
that is not the last.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/keymanager/state/ ./domain/storageprovisioning/state/ ./domain/crossmodelrelation/state/model/ ./domain/removal/state/model/ ./domain/schema/" TEST_ARGS="-count=1 -p 4"
```

2. Break a comparison and rerun to see the new tests fail. For example,
   change `HAVING count(user_public_ssh_key_id) = 0` to `= 99` in
   `domain/keymanager/state/state.go`, or `WHERE  0 = (SELECT COUNT(*)` to
   `0 <= (` in `domain/schema/modeltriggers.go`.

3. Check that no `==` remains in production SQL. This lists nothing.

```sh
git grep -n -E '^\s*(AND|OR|WHERE|HAVING|JOIN|LEFT JOIN|ON)\s.*==|\sON\s.*==' -- '*.go' ':!*_test.go' '*.sql'
```

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
