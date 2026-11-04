# refactor(database): replace sqlite iif expressions

Replace the SQLite-specific `IIF` expressions in the model connection
information query and the controller schema views with `IS NOT NULL`.
These predicates already produce the required boolean values and work
with PostgreSQL as well as SQLite.

The connection query and the views retain their existing joins and
filters. Controller-model, controller-cloud and migration flags remain
true exactly when their respective joined rows exist. This is a
portability change, not a fix for a known Dqlite defect.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- ~[ ] Comments saying why design decisions were made~
- ~[ ] Go unit tests, with comments saying what you're testing~
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

Build and install this branch, and bootstrap two controllers on an
available local LXD cloud:

```sh
make install
juju bootstrap localhost iif-source
juju bootstrap localhost iif-target
juju add-model -c iif-source iif-test
```

List the models on the source controller:

```sh
juju models -c iif-source --format yaml
```

The controller model should report `is-controller: true`, and `iif-test`
should report `is-controller: false`.

Migrate the model to the target controller:

```sh
juju migrate iif-source:iif-test iif-target
```

Migration starts asynchronously. Follow its progress with
`juju status -m iif-source:iif-test`; once it completes, check the
destination:

```sh
juju models -c iif-target --format yaml
juju show-model iif-target:iif-test
```

The target should list `iif-test` as a non-controller model, and
`show-model` should return its details.

## Documentation changes

None.

## Links

None.
