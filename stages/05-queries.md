# Stage 05: shared SQL portability and correctness

Branch `pg-poc/05-queries`, on `pg-poc/04-schemagen`.

## Status and scope

Implemented baseline; remaining production tasks below. X33–X36 are
finished according to the latest run; X37/X38 and U11 remain open.
The [commit inventory](../history/compatibility.md#stage-5-inventory) is
historical. Stage 6 supplies the existing PostgreSQL harness; do not
reimplement it. Latest failures are in [its evidence table](06-compatibility.md#latest-recorded-evidence).

Make shared SQL valid while preserving intended behaviour. Classify real
behaviour changes explicitly and retain upstream-ready fixes. Only read
the X*/D* entries for the task being assigned. Follow OPERATIONS.md for
dual-backend tests and PLAN.md for restacking.

## Remaining tasks

### 05-X37: equality syntax

- Read X37. Entry points: `domain/keymanager/state/state.go`
  `deleteUnusedUserKeys`, `domain/storageprovisioning/state/volume.go`
  machine join, and `domain/crossmodelrelation/state/model/offer.go`
  scope predicate.
- Replace SQL `==` with `=` at these sites only. This is portability;
  generated PostgreSQL triggers already translate their own `==`.
- Check the affected package tests on both backends, covering deletion,
  the join and scope filtering, not only successful SQL preparation.
  Keymanager had six recorded PG failures from this issue.
- Stop if failures require unrelated semantics or schema changes.

### 05-X38: removal join scope

- Read X38. Entry point: `domain/removal/state/model/storage.go`,
  `GetDetachInfoForStorageAttachment`.
- Confirm the replacement join before implementation. Explicit
  `CROSS JOIN fulfilment` and moving the one-row CTE first are alternatives,
  not two instructions to execute. Preserve cardinality and returned data;
  consider SQLite's CROSS JOIN planner restriction.
- Verify the three recorded PostgreSQL failures and equivalent dqlite
  cases; cover the intended attachment and missing-result paths.

### 05-CMR: cross-model secrets

Required before the functional milestone (D40), not incidental syntax
cleanup. Read [the dedicated design/fix task](../investigations/crossmodel-secrets.md).
Approve the consumer/revision semantics first, then implement multi-unit
regressions and dual-backend verification. Keep the fix independently
extractable for 4.1 (U11). Design analysis can be parallel; N11 permits an
upstream-only implementation worktree ([branch plan](../UPSTREAM-PLAN.md)).
Shared PoC edits/restacks and PostgreSQL runs remain serial.

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
