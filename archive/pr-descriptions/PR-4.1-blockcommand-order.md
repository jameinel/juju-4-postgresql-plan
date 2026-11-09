# fix(blockcommand): order blocks by type instead of rowid

`GetBlocks` in `domain/blockcommand/state` used `ORDER BY rowid`. `rowid` is an
implicit SQLite column, not standard SQL. Blocks came back in the order they
were enabled, and disabling then re-enabling a block moved it to the end.

This orders by `block_command_type_id` instead. A model has at most one block
of each type, so the order is deterministic: destroy, remove, then change.

This changes what users see. The Block facade `List` call and
`juju disabled-commands` for one model now list blocks in type order instead
of enabling order. `--all` already sorts, so it is unchanged. Other
`GetBlocks` callers don't depend on the order.

New tests enable blocks out of type order, and disable then re-enable one,
and check the type order each time.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Run the unit tests with Dqlite enabled.

```sh
make run-go-tests TEST_PACKAGES="./domain/blockcommand/... ./apiserver/facades/client/block/ ./cmd/juju/block/ ./apiserver/facades/client/controller/ ./apiserver/common/" TEST_ARGS="-count=1 -p 4"
```

2. Enable blocks out of type order and list them. Expect `destroy-model`,
   `remove-object`, `all`. Before this change the order was `all`,
   `destroy-model`, `remove-object`.

```sh
$ juju bootstrap lxd qa-blocks
$ juju add-model blocks
$ juju disable-command all "change first"
$ juju disable-command destroy-model "destroy second"
$ juju disable-command remove-object "remove third"
$ juju disabled-commands
```

3. Re-enable the destroy block and list again. `destroy-model` stays first.

```sh
$ juju enable-command destroy-model
$ juju disable-command destroy-model "destroy again"
$ juju disabled-commands
```

## Documentation changes

None.

## Links

**Issue:** none

**Jira card:** none
