> Proposed replacement description for juju/juju#23731, based on published
> head `76b4a04f09`. This local draft has not been published. The functional
> QA steps below have not been performed.

Destination: https://github.com/juju/juju/pull/23731
Action: replace the PR description, subject to approval.
Title: fix(sequence): target namespace conflicts when allocating sequence values

Both sequence allocators use an upsert without a conflict target, which
PostgreSQL does not accept for `DO UPDATE`. Their unqualified `value`
expression is also ambiguous between the existing row and the excluded row.

Target the namespace primary key and qualify the stored value. Share a scalar
allocator between `NextValue` and `NextNValues`, and use `RETURNING` to read
the allocated value from the upsert without a separate `SELECT`. This keeps
zero-based allocation, contiguous batches, namespace isolation and rollback
within the caller's transaction. A zero-count batch remains a no-op.

Add regressions for mixed and concurrent allocations, namespace variants,
rollback, suppressed writes, and continuation after sequence import. The
fixtures exercise the allocation contract on SQLite/Dqlite; they do not
establish PostgreSQL SQL acceptance or execution.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

On a Linux host with LXD configured and the Juju build dependencies installed,
build this branch and bootstrap a disposable controller using its agent:

```sh
make install
export PATH="${GOBIN:-$(go env GOPATH)/bin}:$PATH"
juju bootstrap localhost sequence-qa-controller --build-agent
juju add-model sequence-qa
juju deploy juju-qa-dummy-storage sequence-qa --storage multi-fs=2
juju status --storage
```

Wait for `sequence-qa/0` to be active and idle. Confirm that it has two
distinct `multi-fs` storage instances. Add two units and inspect the model:

```sh
juju add-unit sequence-qa -n 2
juju status --storage
juju storage
```

Wait for the new units to be active and idle. Confirm that their names are
`sequence-qa/1` and `sequence-qa/2` and that the storage instances have
distinct IDs. Remove one unit, wait until it has disappeared from status,
and add another:

```sh
juju remove-unit sequence-qa/1
juju status --storage
juju add-unit sequence-qa
juju status --storage
```

Confirm that the replacement is `sequence-qa/3`, not a reused unit name, and
that storage IDs remain distinct. Check for sequence allocation errors in
the model log, then destroy the disposable controller:

```sh
juju debug-log --replay --no-tail -m sequence-qa
juju destroy-controller sequence-qa-controller --destroy-all-models --destroy-storage
```

These steps exercise SQLite/Dqlite, not PostgreSQL.

## Documentation changes

No CLI or API workflow changes.

## Links

No associated upstream issue or Jira card.
