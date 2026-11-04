SQLite accepts `IN cte` and `NOT IN cte` as shorthand for membership in a
single-column CTE. Explicit membership subqueries express the same operation
without relying on that SQLite-specific syntax.

Replace eleven expressions with `IN (SELECT column FROM cte)` or its `NOT IN`
equivalent, retaining the existing CTEs and selecting their intended columns.
This preserves endpoint refresh filtering, scoped address and constraint
cleanup, storage lifecycle guards, operation completion and size accounting,
and departed-unit membership. No schema, API or transaction changes.

Add a departure-scoping regression covering empty, partial and full membership,
units scoped to another relation, and units belonging to another application.
It passes with both the original and rewritten SQL on SQLite/Dqlite.

This independently targets `4.1`. The operation-size review #23693 also touches
`prune.go`, but owns different expressions and is not a prerequisite.

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
make juju
juju bootstrap localhost cte-qa --build-agent
juju add-model cte-qa
juju deploy juju-qa-test --revision 22 --channel 2.0/stable --base ubuntu@20.04
juju deploy juju-qa-action
juju deploy juju-qa-dummy-source
juju deploy juju-qa-dummy-sink
```

Wait until the units are idle without hook errors (`juju status --watch 5s`).
Then exercise charm refresh, task completion, relation setup and teardown:

```sh
juju refresh juju-qa-test --revision 23 --channel 2.0/edge
juju integrate dummy-source dummy-sink
juju run juju-qa-action/0 fortune length=long
juju remove-relation dummy-source dummy-sink
juju remove-application dummy-sink
```

Wait for refresh and removal to finish. Check `juju status` shows revision 23,
the action completes with output, and the relation and removed application
disappear without errors in `juju debug-log`. Destroy the disposable
controller after inspection with `juju destroy-controller cte-qa --destroy-all-models`.
This smoke exercise does not cover every changed query (notably storage
attachments, Kubernetes addresses and cross-model departed units).

The four affected state-package race suites passed on SQLite/Dqlite, including
after rebasing onto current `4.1`. A client built from the rebased branch also
ran `juju version`.
Focused regressions passed before and after the rewrite, including the new
departure-scope case. `make pre-check`, Go static analysis and formatting
checks passed on the original branch. The departure-scope stress run completed
38 iterations with zero reported failures before its outer timeout.

The manual smoke steps above, PostgreSQL execution, the full repository suite
and live migration QA were not run. The change makes these membership
expressions portable; it does not claim the affected packages are otherwise
PostgreSQL-compatible.

## Documentation changes

None. No user-facing workflow changes.

## Links

Related independent operation-size review: #23693.
