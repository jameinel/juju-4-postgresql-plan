**Title:** `fix(crossmodelrelation): report secrets when any remote unit is behind`

**Branch:** `4.1-crossmodel-secret-revisions` (base `upstream/4.1` @ `04bc64ef54`)
**Commits:** `6068fa451a` (fix), `4992247076` (refactor)
**Target:** `juju/juju` `4.1`
**PR:** https://github.com/juju/juju/pull/23688

---

The offering model watches secret revisions on behalf of consuming models,
so they can tell their units that a consumed secret has changed. Two
queries in `domain/crossmodelrelation/state/model/secrets.go` feed the
watcher:

- `InitialWatchStatementForRemoteConsumedSecretsChangesFromOfferingSide`
  gives the revision UUIDs for the initial event;
- `GetRemoteConsumedSecretURIsWithChangesFromOfferingSide` maps a batch of
  changed revision UUIDs (including the initial batch) to secret URIs.

Both grouped by secret and tested a bare, ungrouped
`sruc.current_revision` against `MAX(sr.revision)`. With more than one
remote consuming unit, SQLite takes the bare column from an arbitrary row
holding the maximum. If that unit already had the latest revision, the
secret was dropped, even when another unit was behind (see below for the
effect on consuming units). The queries also used the SQLite-only `instr`
function and are rejected by PostgreSQL (SQLSTATE 42803 for the bare
grouped columns).

Semantics after this change:

- A secret is reported when **any** remote consuming unit of the offered
  application is behind a revision of it.
- The initial query returns the **latest** revision UUID of each such
  secret, once, however many units are behind.
- With a revision filter, a secret is reported only if a filtered revision
  is newer than the revision consumed by some unit. Events for older
  revisions (for example obsolete flags) do not report a secret every unit
  has already moved past.

This does not change when `secret-changed` runs. The consuming model still
compares each unit's tracked revision with
`secret_reference.latest_revision`. These queries only decide whether the
offering model reports a newer revision to raise `latest_revision`.
Publishing a new revision was already reported, as every unit is behind
it. The loss was in catch-up: after a restart or reconnect, the initial
event is the only path that raises `latest_revision`, as a consuming
unit's own `secret-get --refresh` does not. Units that were behind
therefore missed `secret-changed` until the next revision. Units that are
already current get no extra hooks.

Units are matched to their synthetic application with
`substr(unit_name, 1, length(app.name) + 1) = app.name || '/'`, which is
exact (so `remote-foo` never matches `remote-foo-bar/0`) and portable. The
`unit` table is not joined, as remote consumer records can outlive the
synthetic units they name.

A second commit applies the same rewrite to the local-consumer queries in
`domain/secret/state` (`InitialWatchStatementForConsumedSecretsChange`,
`GetConsumedSecretURIsWithChanges`). Those are restricted to one unit, so
the bare columns were well defined on SQLite/dqlite; the rewrite keeps the
results identical and makes the SQL valid on PostgreSQL. Its new test
passes against both the old and new queries.

Out of scope, for a separate change: the watcher is scoped by the offered
application (`application_remote_consumer.offerer_application_uuid`)
rather than by the consuming offer connection, so the rows considered
include remote units of every consumer of the offer. Scoping it per
connection touches the facade, service and consuming worker (which starts
one secrets watcher for its first relation only).

## Checklist

- [x] Code style: imports ordered (`gci`), good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests~ (state-level fix; covered by state and watcher tests)
- [ ] ~doc.go added or updated in changed packages~

## QA steps

Unit tests (SQLite, untagged):

```sh
go test -race -count=1 ./domain/crossmodelrelation/... ./domain/secret/...
golangci-lint run --new-from-rev=HEAD ./domain/crossmodelrelation/... ./domain/secret/state/...
```

New tests in `domain/crossmodelrelation/state/model/secrets_test.go`:

- no consumers; single unit current and behind;
- mixed unit revisions in all four orderings of "which unit is behind" ×
  "which consumer is recorded first";
- several secrets and consuming applications, latest revision returned
  once;
- revision filters: only consumed revisions, a revision newer than one
  unit, the latest revision, every revision, and unrelated secrets;
- application name prefixes (`remote-foo` / `remote-foo-bar`) across two
  offered applications.

`TestWatchRemoteConsumedSecretsChangesMixedUnitRevisions` in
`domain/crossmodelrelation/watcher_test.go` starts the watcher with one
unit current and another behind, and checks the initial event and a
later revision.

Against the previous queries, the mixed-unit, latest-revision and filter
state tests fail (two of the four mixed orderings are lost), and the
watcher test never receives its initial event.

The old and new SQL were also run on PostgreSQL 16 against minimal
tables: the old remote query fails (`instr` missing), the old local query
fails with SQLSTATE 42803, and the new queries return the expected rows.

Manual QA (optional sanity check; the regression itself is only reliably
reproduced by the tests above): offer an application that grants a secret
to a relation, consume it from another model with two units, publish a new
revision, and check that both consuming units run `secret-changed`.

## Documentation changes

None.

## Links

**Issue:** n/a
**Jira card:** n/a
