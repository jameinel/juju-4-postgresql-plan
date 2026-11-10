# PostgreSQL PoC: Stage 6 worker W4, timestamps and test literals (2026-11-10)

## Objective

Repair 82 failed leaf tests in eight packages recorded by the
2026-11-10 re-sweep at `09=a3818782aa`. Most are X27 timestamp-precision
round trips. The rest are test SQL that uses `%q` (X6), one unordered
watcher assertion (X17) and one out-of-range timestamp bound. All are
test-side.

Follow the
[Worker protocol](2026-11-10-stage-6-parallel-repairs.md#worker-protocol)
for workspace, scope, test commands, commits and the report.

## Starting state

- The integrator gives you the worktree (`../juju-4-pg-w4`), branch
  (`pg-work/w4-timestamps`), base commit and `-p` budget.
- Evidence: [06-compatibility.md](../stages/06-compatibility.md),
  [DIALECT.md X27](../DIALECT.md) and the sweep JSONL named in the
  [parallel-repairs handoff](2026-11-10-stage-6-parallel-repairs.md#starting-state).

## Work and boundaries

### X27 rule

PostgreSQL `timestamptz` stores microseconds, and pgx truncates
parameters too. go-dqlite round-trips nanoseconds. Make each fixture
time microsecond-representable where the test creates it, for example
`time.Now().UTC().Truncate(time.Microsecond)`. Do not loosen equality
to `tc.Almost` or truncate the value read back: two different
nanosecond instants can map to the same stored value, so that would
hide ordering bugs. Keep boundary assertions (`<`, `<=`, expiry
cut-offs) exact. If a time comes from production code (`clock.Now()`
inside a service or state method), stop that test and report it.

Upstream home for X27 commits: `new` (UPSTREAM-FIXES.md "Unassigned
test findings"). One commit per package is enough.

### Items

1. **`domain/status/state/model`, 49 tests (X27).** 34 report
   `Since` mismatches and 15 report a sub-microsecond
   `time.Duration` difference, in `modelstate_test.go` (37),
   `storage_test.go` (8) and `crossmodelrelation_test.go` (4). There
   are about 110 `time.Now()` calls in these files; fix the shared
   fixture helpers first, then the per-test values.
2. **`domain/secretbackend/state`, 11 tests (X27).** Ten fail at
   `state_test.go:289` (`actualNextRotationTime.Time.Equal(...)` in a
   shared assertion helper) and one at :1793
   (`TestGetSecretBackendRotateChanges`). Fix the expected times where
   the tests create them.
3. **`domain/secret/state`, 8 tests (X27).** `TestGetRotationExpiryInfo`
   (:480), `TestListCharmSecretsApplicationOrUnit` (:1439),
   `TestUpdateSecretContent` (:2449), `TestUpdateSecretContentNonUTCInput`
   (:2502), `TestUpdateSecretContentObsolete` (:2550),
   `TestGetSecretsRotationChanges` (:4409),
   `TestGetSecretsRevisionExpiryChanges` (:4569) and `TestSecretRotated`
   (:4676, exact `Equal` at :4698). The last three already use
   `tc.SameContents`; only the times differ. Keep
   `TestUpdateSecretContentNonUTCInput` testing a non-UTC input: truncate
   without converting to UTC.
4. **`domain/macaroon/state`, 6 tests.** `rootkeys_test.go:30` sets the
   package-level `now = time.Now()`.
   - `TestInsertAndGetKey` (:68), `TestFindLatestKeyExpiresBefore`
     (:130) and `TestFindLatestKeyEquality` (:147) fail in
     `compareRootKeys` (:251-256). X27: truncate `now`.
   - `TestFindLatestKeyReturnsMostRecent` (:97),
     `TestFindLatestKeyExpiresAfter` (:107) and
     `TestFindLatestKeyCreatedAfter` (:117) return "root key not found".
     All pass `time.Unix(math.MaxInt64, 0)` as `expiresBefore`. That is
     beyond the PostgreSQL `timestamptz` range (294276 AD), and pgx
     appears to wrap it silently instead of failing. Confirm the cause
     (for example by logging the bound read back with `SELECT $1::timestamptz`
     in a scratch test), then use a representable far-future bound.
     Production passes `now + expiry` (`domain/macaroon/service/rootkeys.go:87`).
     Record it as `Dialect: new` (out-of-range timestamp parameter).
5. **`domain/operation`, 1 test (X27).** `TestWatchTaskLogs`
   (`watcher_test.go:212`).
6. **`domain/operation/state`, 5 tests.**
   - X6: `getRowCountByField` (`package_test.go:130`) formats the value
     with `%q`, so PostgreSQL reads it as an identifier (42703). Use a
     bind parameter for the value; keep table and field formatted. This
     fixes `TestAddExecOperationWithMachinesOnly` (`start_test.go:36`),
     `TestAddExecOperationEmptyTarget` (:169),
     `TestAddExecOperationOnAllMachinesNoMachines` (:298) and
     `TestAddActionOperationSingleUnit` (:399).
   - X27: `TestImportOperationsWithSingleUnitTask` (`migration_test.go:48`).
     If its cause is not timestamp precision, report it.
7. **`domain/modelmigration/state/model`, 1 test (X6).**
   `TestGetMachineInstanceIDs`: `state_test.go:135` builds
   `INSERT INTO availability_zone VALUES(%q, 'az-1')`. Use a bind
   parameter.

   Items 6 and 7 are follow-ups to U6 (`%q` in test SQL;
   juju/juju#23696 is merged). Use `Upstream-Home: new` and say "U6
   follow-up" in the commit body.
8. **`domain/secret`, 1 test (X17).** `TestWatchSecretsRevisionExpiryChanges`
   (`watcher_test.go:880`) receives the two changes in the opposite
   order. Production `getSecretsRevisionExpiryChanges`
   (`domain/secret/state/state.go:3928`) has no `ORDER BY`, and
   `watchertest.SecretTriggerSliceAssert` compares in order. Check
   whether the secret expiry worker depends on the order of a batch. If
   it does not, make this test's assertion order-insensitive locally
   (for example a test-local assert that sorts by URI before comparing).
   Do not change the shared `SecretTriggerSliceAssert`. If it does,
   report it.

## Tests

The packages are small except `domain/secret` and
`domain/status/state/model`; run each package fully on both backends
with the protocol's commands. Run the X27 packages twice on PostgreSQL
(`-count=2`) to catch fixtures that pass only when the clock happens to
land on a whole microsecond.

## Safety and stop conditions

- Do not change production code, schema, `core/watcher/watchertest` or
  shared test packages outside the listed packages.
- Stop a test and report it if its time comes from production code, or
  if a fix needs a looser comparison.
