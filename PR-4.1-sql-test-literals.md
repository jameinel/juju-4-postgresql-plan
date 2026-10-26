# refactor: use single-quoted string literals in sql

SQL in 51 test files and two production queries wrote string values in double
quotes, for example `VALUES (-1, "dummy")` or `DATETIME("now")`. In standard
SQL double quotes delimit an identifier. SQLite only reads `"x"` as a string
when no column is named `x`, and rejects it when built with `SQLITE_DQS=0`.
This changes those values to single quotes. The values themselves are
unchanged. Double-quoted identifiers such as `"key"` are left alone.

There are three commits:

1. 45 test files.
2. Two production queries: `strftime('%s')` in `insertSchemaVersion` in
   `core/database/schema`, and `DATETIME('now')` in upgrade
   `SetControllerDone`. Neither is part of a hashed schema patch. New tests
   check the values they record.
3. Six more test files, kept separate because other open PRs also edit them.
   Each of those PRs still merges cleanly with this one.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Check that only quote characters changed. With the branch checked out,
   this prints nothing.

```sh
check() {
  diff <(git diff -U0 "$@" | grep -E '^-[^-]' | cut -c2- | tr -d "'\"\\\\") \
       <(git diff -U0 "$@" | grep -E '^\+[^+]' | cut -c2- | tr -d "'\"\\\\")
}
check HEAD~3 HEAD~2
check HEAD~2 HEAD~1 -- core/database/schema/query.go domain/upgrade/state/state.go
check HEAD~1 HEAD
```

2. Run the unit tests of the changed packages, and of the packages that use
   the changed helpers in `domain/model/state/testing` and
   `domain/cloud/state/testing`, with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/access/ ./domain/access/state/ ./domain/agentpassword/state/ ./domain/annotation/state/ ./domain/application/ ./domain/autocert/state/ ./domain/cloud/state/ ./domain/controllerupgrader/state/ ./domain/credential/state/ ./domain/crossmodelrelation/service/ ./domain/crossmodelrelation/state/model/ ./domain/externalcontroller/state/ ./domain/keyupdater/state/ ./domain/unitstate/state/" TEST_ARGS="-count=1 -p 4"
make run-go-tests TEST_PACKAGES="./domain/machine/ ./domain/modelimport/state/model/ ./domain/network/ ./domain/network/state/ ./domain/port/ ./domain/port/state/ ./domain/removal/ ./domain/removal/state/model/ ./domain/schema/ ./domain/secret/ ./domain/status/ ./domain/status/state/model/" TEST_ARGS="-count=1 -p 4"
make run-go-tests TEST_PACKAGES="./domain/storage/state/ ./domain/storageprovisioning/ ./domain/storageprovisioning/state/ ./internal/worker/dbaccessor/ ./domain/model/state/model/ ./domain/model/state/controller/ ./domain/keymanager/ ./domain/keymanager/state/ ./domain/model/bootstrap/ ./domain/modelconfig/ ./domain/modelconfig/bootstrap/ ./domain/modeldefaults/state/ ./domain/secretbackend/ ./domain/status/state/controller/ ./internal/migration/" TEST_ARGS="-count=1 -p 4"
make run-go-tests TEST_PACKAGES="./core/database/schema/ ./domain/upgrade/state/ ./domain/crossmodelrelation/ ./domain/modelimport/" TEST_ARGS="-count=1 -p 4"
```

3. Change `strftime('%s')` to `strftime('%Y')` in
   `core/database/schema/query.go` and `DATETIME('now')` to
   `DATETIME('now', '-1 hour')` in `domain/upgrade/state/state.go`, then
   rerun the last command. `TestEnsureRecordsUpdatedAt` and
   `TestSetControllerDoneRecordsCompletedAt` fail.

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
