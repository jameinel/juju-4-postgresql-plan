# fix(database): bind controller node id as decimal text

The `controller_node.dqlite_node_id` column is `TEXT` and holds the Dqlite
`uint64` node ID. It is text rather than `INT` because Dqlite node IDs are
derived from a SHA hash, so they span the full unsigned 64-bit range and often
have the high bit set. SQLite integers are signed 64-bit, so an `INT` column
cannot represent those values. `database/sql` also refuses to bind a `uint64`
with the high bit set (`uint64 values with high bit set are not supported`).

`InsertControllerNodeID` bound the `uint64` directly, so it fails for any ID
with the high bit set. This is a latent defect rather than an observed failure:
its only production caller is `BootstrapDqlite`, and the bootstrap node always
gets go-dqlite's fixed `BootstrapID` (`0x2dc171858c3155be`), which is below
2^63. Nodes that join later get hash-generated IDs, but they are recorded by
`controllernode` `State.AddDqliteNode`, which already formats the ID as a
decimal string for exactly this reason. For IDs below 2^63, SQLite's TEXT
affinity already converted the bound integer to the same decimal text, so
existing data is unchanged and no upgrade step is needed.

This change formats the ID with `strconv.FormatUint` before binding, matching
`AddDqliteNode`, so the function is correct for every `uint64` it accepts and
binds a value that matches the column type. The column is deliberately not
changed to a signed `INT`; the schema and function comments now explain why.
Test fixtures in changestream, upgrade and status that inserted integers or
`uint64` values into that column now bind decimal strings, matching
production. The new test covers 0, 42, 1<<63, `0x2dc171858c3155be` and
`MaxUint64`, checking the stored text and `typeof = 'text'`.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled. A plain `go test` skips the Dqlite
   worker suite.

```sh
make run-go-tests TEST_PACKAGES="./internal/worker/dbaccessor/ ./internal/database/ ./domain/upgrade/state/ ./domain/changestream/state/ ./domain/status/state/controller/ ./domain/schema/..." TEST_ARGS="-count=1"
```

2. Check the regression: revert only `internal/database/bootstrap.go`, then run
   `go test ./internal/database/ -run TestBootstrapSuite`.
   `TestInsertControllerNodeIDUnsignedRange` should fail with
   `uint64 values with high bit set are not supported`.

3. Bootstrap a controller and confirm the stored ID is text. `juju_db_repl`
   opens the controller database by default.

```sh
$ juju bootstrap lxd qa-nodeid
$ juju ssh -m controller 0
$ . /etc/profile.d/juju-introspection.sh
$ juju_db_repl
repl (controller)> SELECT controller_id, dqlite_node_id, typeof(dqlite_node_id) FROM controller_node;
```

   Expect controller `0` to have `dqlite_node_id` = `3297041220608546238`
   (`BootstrapID`) with `typeof` = `text`, the same value as before this
   change.

4. Add controller units and confirm the joining nodes, which use
   hash-generated IDs and the existing `AddDqliteNode` path, are stored in the
   same representation.

```sh
$ juju add-unit -m controller controller -n 2
$ juju status -m controller   # repeat until 3 controller units are started
```

   With `juju-controller` 4.1/stable rev 468 the new units currently fail
   `config-changed` because the SSH port push needs the control socket before
   the node has joined Dqlite (introduced by juju/juju-controller#145 and
   juju/juju-controller#153; unrelated to this change). Skip that hook so the
   `dbcluster` hooks can run:

```sh
$ juju resolved --no-retry controller/1 controller/2
`````

   Re-run the query from step 3. Expect three rows, all `text`. Joining IDs
   with the high bit set (above 9223372036854775807) are likely and must
   appear as unsigned decimals, not negative numbers.

5. Check the logs for errors.

```sh
$ juju debug-log -m controller --replay --no-tail --level WARNING
```

6. No facades change, so no model migration testing is needed.

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
