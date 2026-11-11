# Stage 05: shared SQL portability and correctness

Branch `pg-poc/05-queries`, on `pg-poc/04-schemagen`.

## Status and scope

PoC runtime fixes implemented. X33–X38 and U11 are finished (U11 is
cherry-picked from the open upstream PR). Bootstrap runs 2 and 3 and
PostgreSQL sweep 2 (2026-11-05) found more production defects; the runtime
failures are addressed (05-BOOT-FIXES,
05-SECRET-X16, 05-X16-rest, 05-unitstate-relation-key). The lease-clock
regression is fixed at `05=76aac853dd`. U24 and U25 are fixed in the
following two commits; see below.
U21 is a boundary workaround, not a completed shared schema correction;
the integer-schema upstream task and later PoC reconciliation remain open.
The D55 review also holds upstream storage #23766 for attachment-invariant
investigation and rework; lowest-UUID selection is not an established
correct-mount contract. The revised upstream grouped-column implementation
preserves CA certificates as well as fixing SQL. These are upstream
dispositions, not new PostgreSQL validation of this PoC tip.

Validate Stage 5 fixes on PostgreSQL at the 09 tip (or a worktree of 09
with the fix applied): `JUJU_POSTGRESQL_TEST_URL` only redirects tests
once the Stage 6 harness is present, so a PostgreSQL run on 05 alone
silently uses SQLite.
The [commit inventory](../history/compatibility.md#stage-5-inventory) is
historical. Stage 6 supplies the existing PostgreSQL harness; do not
reimplement it. Latest failures are in [its evidence table](06-compatibility.md#latest-recorded-evidence).

Make shared SQL valid while preserving intended behaviour. Classify real
behaviour changes explicitly and retain upstream-ready fixes. Only read
the X*/D* entries for the task being assigned. Follow OPERATIONS.md for
dual-backend tests and PLAN.md for restacking.

Apply D55 to shared findings: correct SQL/schema semantics rather than only
making Go conversions succeed. PoC-only dialect support remains separate.
Use [UPSTREAM-PLAN.md](../UPSTREAM-PLAN.md#purpose-and-sqlschema-acceptance)
for upstream design and acceptance; do not treat these source implementations
as required upstream approaches. Ask before changing an assigned scope.

## Remaining tasks

### Nullable BOOLEAN dqlite workaround (integrated locally)

The refreshed juju/juju#23759 breakout was integrated on Stage 5:
`c9a168b5cd` as `0997c97b74`, `c51a0e05a1` as `0b54417c0b` and
`39b1dc3cd8` as `6630d3b862`. These retain the upstream authors and
messages. The temporary INTEGER projections preserve NULL, false and true
until the bundled dqlite contains canonical/dqlite#883. Stage 6's old
machine-test projection commit `635744b351` was dropped with user approval
as redundant with the first upstream commit.

At `09=3883b6ce94`, focused logging, provisioner, machine-reprovision and
storage-status checks passed with `-race` on SQLite, PostgreSQL and real
dqlite, including the formerly failing nil Loki case. This is not a full
sweep. See [Stage 6 evidence](06-compatibility.md#nullable-boolean-integration-2026-11-11)
for the dqlite-only sentinel and remaining catalog race.

### 05-U24 / 05-U25: SSH conflicts and agent streams (done locally)

- `2f55c00439` (U24, latent/X18) uses targeted conflict-safe inserts and
  reads the persisted machine or unit host key. It preserves the winning key
  and algorithm and propagates other insert errors. Upstream extraction:
  `4.1-ssh-host-key-conflict`.
- `274e1a2e02` (U25, portability/X9) reads only stored binary fields and
  sets the result stream from the caller, preserving the model stream check
  and the controller store's existing behavior. Upstream extraction:
  `4.1-controllerupgrader-stream-result`.

At restacked `09=85e4e6b122`, 13 SSH host-key and seven controllerupgrader
stream methods passed on each of PostgreSQL and real dqlite with `-race`,
without failures or skips. Coverage includes creation, existing and
concurrent keys, retained algorithms, missing entities, invalid algorithm,
two streams, empty and mismatched stores, and invalid versions. A
connection-refused probe confirmed PostgreSQL selection for both packages.
Bounded race stress of the two concurrent host-key cases completed 16
real-dqlite and 10 PostgreSQL runs without failures; one active run was
stopped at each 31-second bound. Logs and compile output:
`~/pg-poc-work/integrator/u24-u25/`. `make pre-check` passed with its
default static-analysis skip, not a full lint run.

After these checks, the user approved correcting the commit trailer blocks.
The commits and later branches were rewritten; the resulting `09=ff21df57ed`
has the same source tree as tested `09=85e4e6b122`.

The upstream breakouts remain separate and unclaimed. The 142-package
final sweep and redistribution of the four Stage 9 checkpoints remain
outstanding; these focused checks do not replace them.

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
  most one cloud instance row. Test SQL fixed in Stage 6 (`fa75871904`, was `283a00f630`).

Both are upstream candidates (UPSTREAM-FIXES.md U20). See the
[Stage 9 run log](09-bootstrap.md#09-functional-run-log).

### 05-X16-rest: remaining bare grouped columns (done)

Found by sweep 2. One commit per domain, so each can go upstream
separately (U22). PostgreSQL at `09=760f3b675f`: no 42803 left in these
packages.

- `52a0eea6ee` secretbackend: `ListSecretBackends` and the in-use
  Kubernetes backend query group by every selected column;
  `getK8sSecretBackendForModel` uses `SELECT DISTINCT`. The extra columns
  are functionally dependent on the old keys, so results are unchanged
  (portability only). Existing tests cover the queries; 11 PostgreSQL
  failures remain, all X27 rotation-time `Equal` checks (06-X27).
- `367f754dd6` removal: the controller copy of
  `getK8sSecretBackendForModel` uses `SELECT DISTINCT` instead of
  `GROUP BY vm.name, vcca.attribute_key`. Added
  `TestGetActiveModelSecretBackendK8sBuiltIn`, which reproduced 42803.
  Package passes on both backends.
- `640d3bcabf` storage: `GetStorageInstanceInfo` joined every filesystem
  attachment on the unit's net node and collapsed them with
  `GROUP BY sa.uuid`, so SQLite returned an arbitrary mount point when two
  existed for the same filesystem and node (a latent bug; the schema does
  not forbid it). The join now picks the lowest attachment UUID, and the
  `ORDER BY` moved out of the `FROM` subquery for both attachment queries.
  `TestStorageInstanceInfoFilesystemDuplicateNodeAttachments` failed 7 of
  10 runs against the old query on SQLite.
- `persistent` (and `in_use` in the import tests) in `domain/storage/state`
  are fixture binds of integer literals to booleans
  (`base_test.go` `newPersistentModelVolume`, `import_test.go`), not
  production: 06-bind-types. The remaining 8 PostgreSQL failures there
  are those 4 binds and 4 X27 time comparisons.

Observation, not changed: both `getK8sSecretBackendForModel` copies
return one CA certificate per cloud even if a cloud has several (the row
converters keep the last row). Behaviour is unchanged from SQLite.

Test-side X16 sites (`application/state/unit_test.go`,
`removal/state/model/application_test.go`) belong to Stage 6 (06-X16).

### 05-unitstate-relation-key: PoC workaround implemented; schema fix pending

Historical implementation: `1afd3b9434`. The relation IDs are bound as
decimal text and parsed back into `int` on read, with an error for a
non-decimal key; the column stays `TEXT`, so there is no schema or import
change. Existing tests covered it
(the 4 failures below); `domain/unitstate/state` passes on PostgreSQL at
`09=760f3b675f`.

Found 2026-11-05 at `05=3a8f0090c2`, after X16 no longer masked it.
`domain/unitstate/state/state.go` `setUnitStateRelation` inserts
`unitRelationStateKeyVal` (`unitStateKeyVal[int]`) into
`unit_state_relation`, whose `key` column is TEXT on both backends. SQLite
stores the integer by type affinity; pgx refuses to encode an `int` as
text (4 failures: `TestSetUnitState*`, `TestUpdateUnitStateRelation`).
The uniter commits relation state on every hook, so this blocks any
related application on PostgreSQL. Also check the read path, which scans
the TEXT key back into `int`.

The earlier preference for Go conversion over schema correction is
superseded by D55 and the user's U21 decision. The upstream task changes
the column to `INT`, regenerates the 4.1 export type and converts legacy
text keys during import, without adding a foreign key. Its schema,
import/export, recovery and unit-state checks are in UPSTREAM-PLAN.md.
The PoC workaround remains historical implementation evidence, not an
approved upstream solution. Reconcile it with the landed result under
UPSTREAM-INTEGRATION.md; do not silently mirror an in-review design here.

### 05-lease-db-time: lease times from the database clock (done)

D16 (amended 2026-11-06). `8930466847` replaced `datetime('now', ...)` in
`domain/lease/state/state.go` (claim, extend) and
`internal/worker/leaseexpiry/store.go` (both expiry queries) with times
from `clock.WallClock` on the calling controller. `leaseexpiry` runs on
every controller, so a controller whose clock runs ahead deletes leases
early. Upstream evaluates all three on the dqlite leader, which is the
behaviour to keep.

`76aac853dd` restores the database clock. Claim and extension read
`CURRENT_TIMESTAMP` inside each transaction attempt, preserve the
whole-second rounding and use a transaction-local lease value on retry.
Both expiry queries compare with `CURRENT_TIMESTAMP`; the state and store
clocks are removed, while the worker's scheduling clock remains.

The deterministic SQLite clock fixture covers controller skew, the
equality boundary, pinning and resampling after a rolled-back attempt.
The controller-schema tests also cover claim, extension and expiry against
real database time, including pinned, null-expiry and future leases.
`CURRENT_TIMESTAMP` is returned as `string` by the SQLite and real-dqlite
drivers and as `time.Time` by pgx; the scanner also accepts `[]byte` and
rejects invalid values (DIALECT.md X45).

Validation on 2026-11-09 at `09=a3818782aa`:

- `go build -v -p 2 ./...` passed.
- Both `./domain/lease/state` and `./internal/worker/leaseexpiry` passed
  with `-race -count=1 -p 2 -timeout 120s -v` on untagged SQLite, on real
  dqlite via `make run-go-tests` (`dqlite,libsqlite3`), and on PostgreSQL.
  No tests were skipped. An invalid PostgreSQL URL made each affected
  database suite fail to connect, proving backend selection. This is not
  a full re-sweep.
- A 31-second, two-process real-dqlite race stress run of the worker and
  store suites at the Stage 5 source committed as `76aac853dd` completed
  48 iterations with no failures before the expected outer timeout.
- The remaining N16 smoke selection was run after the commit with
  `-race -count=1 -p 4 -timeout 120s -v`: `internal/database`,
  `core/database/schema`, `core/watcher/eventsource`,
  `internal/changestream/stream`, `domain/schema`,
  `domain/changestream/state` and `domain/controllernode/state`.
  Untagged SQLite passed. On PostgreSQL, the other packages passed but
  `internal/changestream/stream` had 36 existing `AUTOINCREMENT` fixture
  failures (06-dialect-fixtures). `domain/lease` has no test files.
  PostgreSQL still skips the schema and dqlite-only tests noted in
  [Stage 6](06-compatibility.md#latest-recorded-evidence).
- Scoped `golangci-lint` and `txncheck` reported zero issues, and `gci`
  was run on the changed Go files. `make pre-check` passed with
  `STATIC_ANALYSIS` unset, so it did not run the full static-analysis
  suite.
- Repository-wide `txncheck` reported 11 pre-existing findings at the
  restacked top: captured mutations in `domain/cloud/state/state_test.go`,
  `domain/unitstate/state/commithook_test.go` and
  `internal/database/postgresql/driver_test.go`, and assertions in
  transactions in `internal/database/bootstrap_postgresql_test.go`.
  Those files match the pre-restack `origin/pg-poc/09-bootstrap`; none is
  part of this fix. The Stage 5 run reported the three mutation findings.

The PostgreSQL run also completes 06-leaseexpiry-db. Logs are in the
session's `files/09-lease-{postgresql,sqlite,dqlite}.log`, with the
`09-lease-smoke-{postgresql,sqlite}.log`, stress, lint and
transaction-checker logs alongside them.

Not in scope: `migration.go` `expiry >= date('now')` (date-only; see
UPSTREAM-PLAN.md), and a replicated lease clock.

## Contracts consumers must preserve

- D16 lease times come from the database clock (`CURRENT_TIMESTAMP`), not
  a controller clock, and are stored as UTC `YYYY-MM-DD HH:MM:SS` text on
  SQLite; expiry is rounded up to whole seconds. Stage 8 agent pools must use
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
