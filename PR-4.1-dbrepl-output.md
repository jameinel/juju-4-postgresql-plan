# refactor(database): share dbrepl change output

The `.change-log` and `.change-stream` commands duplicate row scanning
and tabular output. Use a shared implementation so formatting and error
handling cannot drift independently.

Each command retains its existing SQL, including the change-stream
lower bound, per-key coalescing and ID ordering. Output columns, transaction
retry handling and error reporting remain unchanged.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- ~[ ] Comments saying why design decisions were made~
- [x] Go unit tests, with comments saying what you're testing
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

From this checkout, using a client built from the same source, bootstrap a
disposable controller on an available local LXD cloud:

```sh
juju bootstrap localhost dbrepl-qa --build-agent
juju add-model -c dbrepl-qa dbrepl-test
juju ssh -m dbrepl-qa:controller 0
```

On the controller machine, start the REPL, switch to the new model and
inspect both commands:

```text
juju_db_repl
.switch model-dbrepl-test
.change-log
.change-stream
```

Both commands should display the same five column headers: `id`,
`edit_type`, `namespace`, `changed` and `created_at`. If entries are
present, `.change-log` displays all retained entries; `.change-stream`
displays the latest positive-ID entry for each namespace and changed key,
ordered by ID. Empty results should still show the headers. Change-log
entries can be pruned while inspecting them, so results from separate
commands are not necessarily the same snapshot.

Exit the REPL and SSH session:

```text
.exit
exit
```

Remove the disposable controller when finished:

```sh
juju destroy-controller dbrepl-qa --destroy-all-models --destroy-storage
```

## Documentation changes

None; the commands and their output are unchanged.

## Links

Related: https://github.com/juju/juju/pull/23730
