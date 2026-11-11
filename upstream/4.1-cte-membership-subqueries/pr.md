SQLite accepts `IN cte` and `NOT IN cte` as shorthand for membership in a
single-column CTE. Explicit membership subqueries express the same operation
without relying on that SQLite-specific syntax.

Replace eleven expressions with `IN (SELECT column FROM cte)` or its `NOT IN`
equivalent, retaining the existing CTEs and selecting their intended columns.
This preserves endpoint refresh filtering, scoped address and constraint
cleanup, storage lifecycle guards, operation completion and size accounting,
and departed-unit membership. The `NOT IN` rewrites retain the original
NULL behavior rather than changing the predicates to `NOT EXISTS`. There are
no schema, API or transaction changes.

Cover departure scoping for empty, partial and full membership, units scoped to
another relation, and units belonging to another application. Also check that
deleting one application preserves another application's device constraint
attributes.

This independently targets `4.1`. The operation-size review #23693 also
touches `prune.go`, but owns different expressions and is not a prerequisite.
This change makes these membership expressions portable; it does not claim
the affected packages are otherwise PostgreSQL-compatible.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

On a development machine with LXD and the dependencies for building Juju,
from this checkout install the client and bootstrap a disposable controller
with an agent built from the same checkout:

```sh
export GOBIN="$(mktemp -d)"
export PATH="$GOBIN:$PATH"
make go-install
juju bootstrap localhost cte-qa
juju add-model cte-qa
juju deploy juju-qa-test --revision 22 --channel 2.0/stable --base ubuntu@20.04
juju deploy juju-qa-action
juju deploy juju-qa-dummy-source
juju deploy juju-qa-dummy-sink
```

Wait until the units are idle without hook errors (`watch juju status`).
Then exercise charm refresh, task completion, relation setup and teardown:

```sh
juju refresh juju-qa-test --revision 23 --channel 2.0/edge
juju integrate dummy-source dummy-sink
juju run juju-qa-action/0 fortune length=long
juju status
juju remove-relation dummy-source dummy-sink
juju remove-application dummy-sink
```

Wait for refresh and removal to finish. Check `juju status` shows revision 23,
the action completes with output, and the relation and removed application
disappear without errors in `juju debug-log`. Destroy the disposable
controller after inspection:

```sh
juju destroy-controller cte-qa --destroy-all-models
rm -rf "$GOBIN"
```

This smoke exercise does not cover every changed query (notably storage
attachments, Kubernetes addresses and cross-model departed units).

## Documentation changes

None. No user-facing workflow changes.

## Links

Related independent operation-size review: #23693.
