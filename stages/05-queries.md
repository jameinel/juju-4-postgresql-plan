# Stage 05: shared SQL portability and correctness

Branch `pg-poc/05-queries`, on `pg-poc/04-schemagen`.

## Status and scope

Implemented baseline; remaining production tasks below. X33–X38 and U11
are finished (U11 is cherry-picked from the open upstream PR). Bootstrap
runs 2 and 3 and PostgreSQL sweep 2 (2026-11-05) found more production
defects; those on the bootstrap path are fixed (05-BOOT-FIXES,
05-SECRET-X16). Open: 05-X16-rest and 05-unitstate-relation-key.
The [commit inventory](../history/compatibility.md#stage-5-inventory) is
historical. Stage 6 supplies the existing PostgreSQL harness; do not
reimplement it. Latest failures are in [its evidence table](06-compatibility.md#latest-recorded-evidence).

Make shared SQL valid while preserving intended behaviour. Classify real
behaviour changes explicitly and retain upstream-ready fixes. Only read
the X*/D* entries for the task being assigned. Follow OPERATIONS.md for
dual-backend tests and PLAN.md for restacking.

## Remaining tasks

### 05-X37: equality syntax (done)

`0703122e2a`. Three production `==` predicates became `=`; added a
keymanager test for unused-key cleanup, which had no coverage, and checked
it with mutations. Existing volume join and offer scope tests cover the
other sites. Both backends pass at `08=7dc5b5bbc9` (06-compatibility.md).

### 05-X38: removal join scope (done)

`18d679a2b7`. `CROSS JOIN fulfilment` was chosen: the CTE is a single-row
aggregate, so the join is a deliberate cross join and cardinality is
unchanged. Added a zero-fulfilment test. The detach-info tests pass on both
backends; the full removal package rerun is part of the D42 gate.

### 05-CMR: cross-model secrets (done)

Implemented upstream as juju/juju#23688 (open). Both commits were
cherry-picked unchanged onto this branch as `8f2c49c355` and `79f2e66049`,
so that the PoC can continue before review finishes. Keep them unmodified,
so that reconciliation after landing drops them cleanly. The semantics and
evidence are in [the task](../investigations/crossmodel-secrets.md) and in
U11 in [UPSTREAM-FIXES.md](../UPSTREAM-FIXES.md).

### 05-SECRET-X16: bare grouped columns in secret state (done)

`5e70fcbc84`. The five `domain/secret/state/state.go` sites (`updateSecret`,
`GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets`,
`getSecretsRevisionExpiryChanges`) select the latest revision row with a
correlated `MAX(revision)` subquery instead of SQLite's bare-column rule.
`latestrevision_test.go` covers each with several revisions. The expiry
watcher applies its optional revision UUID filter inside the subquery.
Found together: `0ed13ff885` orders the revision listing by revision
(X17 class), and `3a8f0090c2` fixes the same pattern in unitstate
`updateCharmSecret` (commithook), which run 3's uniter reached.

PostgreSQL at `05=3a8f0090c2`: no 42803 left in `domain/secret/...` or
`domain/unitstate/...`; remaining failures are test-side (X27, order) or
05-unitstate-relation-key. Keep these commits separate from the U11
cherry-picks; they are the `4.1-secret-latest-revision` candidate
([UPSTREAM-PLAN.md](../UPSTREAM-PLAN.md)).

### 05-BOOT-FIXES: bootstrap-path query fixes (done)

Found 2026-11-05 (U20): the network booleans by PostgreSQL package
validation before run 2, and the agentpassword query by run 2:

- `30e2bca25a` network: `GetMachineAppBindings` and `GetUnitAPIAddresses`
  compared booleans with `0`/`1` (42883); now `FALSE`/`TRUE`. The second
  is on the controller API address path.
- `ada2a852ee` agentpassword: the machine password check selected
  `mci.instance_id` beside `COUNT` without grouping (42803), so machine
  agents could not log in. Now `MAX(mci.instance_id)`; a machine has at
  most one cloud instance row. Test SQL fixed in Stage 6 (`283a00f630`).

Both are upstream candidates (UPSTREAM-FIXES.md U20). See the
[Stage 9 run log](09-bootstrap.md#09-functional-run-log).

### 05-X16-rest: remaining bare grouped columns

Found by sweep 2; not yet fixed. Same treatment as 05-SECRET-X16: confirm
the intended row, rewrite portably, add a test that would distinguish
rows on SQLite, then run the package on both backends. One commit per
domain, so each can go upstream separately.

- `domain/secretbackend/state/state.go`: `ListSecretBackends`,
  `ListSecretBackendsForModel` and `getK8sSecretBackendForModel`
  (13 test failures).
- `domain/removal/state/controller/secret.go`:
  `getK8sSecretBackendForModel` (`GROUP BY vm.name, vcca.attribute_key`).
  Not in the sweep failures; found by reading, so it needs a test first.
- `domain/storage/state/instance.go`: `GetStorageInstanceInfo`
  (`GROUP BY sa.uuid`).
- Also in `domain/storage/state`: "column persistent is of type boolean
  but expression is of type integer". Determine whether production or a
  fixture binds it.

Test-side X16 sites (`application/state/unit_test.go`,
`removal/state/model/application_test.go`) belong to Stage 6 (06-X16).

### 05-unitstate-relation-key: integer key in a TEXT column

Found 2026-11-05 at `05=3a8f0090c2`, after X16 no longer masked it.
`domain/unitstate/state/state.go` `setUnitStateRelation` inserts
`unitRelationStateKeyVal` (`unitStateKeyVal[int]`) into
`unit_state_relation`, whose `key` column is TEXT on both backends. SQLite
stores the integer by type affinity; pgx refuses to encode an `int` as
text (4 failures: `TestSetUnitState*`, `TestUpdateUnitStateRelation`).
The uniter commits relation state on every hook, so this blocks any
related application on PostgreSQL. Also check the read path, which scans
the TEXT key back into `int`.

Prefer converting in Go (bind the key as a decimal string) over changing
the column type, which would need an upstream schema change and
import/export review. An upstream candidate either way: the schema and
the Go type disagree.

## Contracts consumers must preserve

- D16 lease times are bound as UTC `YYYY-MM-DD HH:MM:SS` text; expiry is
  rounded up to whole seconds. Stage 8 agent pools must use
  `WithTimeZoneUTC`, as the harness does (X13). Plain current-time columns
  use `CURRENT_TIMESTAMP`; port calculations without changing meaning.
- PostgreSQL aborts a transaction after a statement error (X18). Do not
  catch a constraint error and continue using that transaction. An
  existence check alone does not universally guarantee a concurrent
  conflict becomes a retryable serialization failure.
- Stage 4 owns generated DDL; Stage 6 owns general test-fixture repairs.
  Shared production fixes and their regression tests belong here.

## Deferred, not forgotten

PostgreSQL model import/export (D40), including X21 FK deferral, is not
required for the first functional comparison. Recovery remains dqlite-only.
Track these gaps in DIALECT.md and Stage 11; shared 4.1 schema changes
still require upstream import/export compatibility coverage.
