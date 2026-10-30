SQLite does not make an `INT PRIMARY KEY` column implicitly non-null.
These lookup IDs are supplied explicitly, but 48 model tables and 17
controller tables accepted NULL and generated nullable export IDs.

Declare those 65 keys `INT PRIMARY KEY NOT NULL` and regenerate the 4.1
export types and model transforms. Keep `INTEGER PRIMARY KEY` unchanged:
that exact SQLite type aliases the rowid and allocates an ID for a NULL
insert. Composite keys and unrelated nullable attributes are out of scope.

Legacy 4.0.12-format lookup conversions preserve complete rows, reject
missing IDs with table/row context, and accept populated zero IDs. Validate
changed, consumed 4.1 YAML IDs before scalar decoding can turn an omitted
or NULL ID into zero. Source changestream lookup rows remain target-local
and excluded from import. Recovery continues to enforce its existing raw
column validation and transactional replay requirements.

This targets upstream `4.1` at `8369e6e2a9` (`4.1-beta3`). Base declarations
are changed before the 4.1 release; historical schemas and export types
are unchanged. Cross-minor migration creates the destination schema rather
than modifying the source controller's applied schema chain.

### Automated evidence

- Race-enabled schema, export, import, recovery, cloud-state and affected
  generator suites pass.
- The scoped schema guard inventories every single-column INT key in both
  schemas and checks all 65 changed export fields, NULL rejection, valid
  seed IDs, and retained INTEGER rowid allocation.
- Complete-row legacy conversion tests cover zero/nonzero/nil IDs,
  multiple rows and nullable attributes. Real persistence round trips
  cover seed conflicts, non-seed rows and changestream exclusions.
- Raw YAML tests cover every changed consumed lookup, omitted/NULL IDs,
  zero IDs and aliases/merges. Generated recovery decode and replay tests
  cover nullable attributes, malformed dumps and rollback without partial
  replay.
- Stress: schema 19, legacy import 16 and recovery 9 completed runs, with
  no failures before the intentional outer timeout.
- Focused build-tagged lint reports zero issues. `make go-build`, import
  formatting and diff checks pass; `make rebuild-export` is idempotent.
  `make pre-check` passes but skips static analysis by default; the focused
  lint run above supplies that check.

Live bootstrap, deployment and migration QA below has **not yet been run**.
No PostgreSQL validation is claimed.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [x] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages

## QA steps

Use disposable controllers and branch-built CLI/agents. The migration
source should be a supported 4.0 controller using the 4.0.12 export format.

### 1. Fresh 4.1 bootstrap and deployment

```sh
juju bootstrap lxd lookup41 --build-agent
juju add-model -c lookup41 fresh
juju deploy -m lookup41:fresh juju-qa-test qa -n 2
juju status -m lookup41:fresh
```

Verify active/idle units. Change config, run an action and add/remove a
unit; confirm normal convergence and successful action results.

### 2. Migrate a populated 4.0 model

Prepare a healthy `lookup40:move-me` with multiple units, a relation,
attached storage, a test secret and non-default charm config. Record the
state and workload data, then migrate:

```sh
juju status -m lookup40:move-me
juju migrate lookup40:move-me lookup41
juju status -m lookup41:move-me
```

Verify that config, relations, storage/data and secret contents survive,
and the workload returns to active/idle.

### 3. Exercise the migrated model

```sh
juju deploy -m lookup41:move-me juju-qa-test post-migration -n 2
juju status -m lookup41:move-me
juju add-unit -m lookup41:move-me post-migration
# Wait for post-migration/2 to become active/idle before removing it.
juju remove-unit -m lookup41:move-me post-migration/2
```

Change config and run an action on a migrated application. Verify normal
convergence and continued relation, storage and secret access.

### 4. Check 4.1 re-export and decoding

```sh
juju bootstrap lxd lookup41b --build-agent
juju migrate lookup41:move-me lookup41b
juju status -m lookup41b:move-me
```

Repeat the preservation checks and a config/action or scaling operation.
This exercises the new scalar-ID export and 4.1 YAML decoder, rather than
only the legacy decoder.

Check logs, including the migration period. Repeat the first two commands
with `lookup41b` for the second destination:

```sh
juju debug-log -m lookup41:controller --replay
juju debug-log -m lookup41:move-me --replay
juju debug-log -m lookup41:fresh --replay
```

Expect no new lookup-ID, payload-decoding, NOT NULL, foreign-key or
duplicate-seed errors, and no stalled agents/status updates. Missing/NULL
ID rejection and recovery rollback are covered by targeted regressions.

## Documentation changes

Updated schema package documentation to distinguish explicit non-null INT
lookup keys from INTEGER rowid allocation. Normal CLI workflows do not
change. Malformed consumed lookup rows with omitted/NULL IDs are rejected
instead of silently accepting NULL or inventing zero.

## Links

- [SQLite PRIMARY KEY semantics](https://www.sqlite.org/lang_createtable.html#the_primary_key)
- Related, independent composite-key work: juju/juju#23710 and
  juju/juju#23723. This branch does not include or depend on those fixes.
  When the exposure and lookup changes are integrated, regenerate from the
  combined schema and remove the global primary-key guard's INT exemption,
  retaining only the INTEGER rowid exemption. Run both scoped/global guards.

~Issue: no linked issue.~

~Jira card: no linked Jira card.~
