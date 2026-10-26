# Stage 05: shared SQL portability and correctness

Branch `pg-poc/05-queries`, on `pg-poc/04-schemagen`.

## Status and scope

Implemented baseline; remaining production tasks below. X33–X38 are
finished; U11 remains open.
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
