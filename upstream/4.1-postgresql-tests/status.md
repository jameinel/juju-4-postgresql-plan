# 4.1-postgresql-tests

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Infrastructure-only branch at `bd970f693c`; implementation has not started
in the recorded state. Backend design remains proposed, not approved.
This is separate test-infrastructure work, not a stock SQL/schema fix;
see the [PoC scope](../../PLAN.md#postgresql-sql-regression-branch).

<!-- Migrated from PLAN.md; historical evidence retains its revision. -->

#### Infrastructure workflow discussion (2026-11-11)

Local branch `4.1-postgresql-tests` was created at upstream 4.1
`bd970f693c`. Its worktree is
`../4.1-postgresql-tests`, alongside the PoC and planning repositories
under the primary checkout's locally Git-ignored `.worktrees/`. It links
`_deps` to the primary checkout's `../../_deps`; the PoC retains its own
local copy. Future worktrees for this effort also belong under
`.worktrees/`. No implementation commits or pending upstream patches have
been added. The PoC's tracked files remain unchanged.

The confirmed scope is a branch directly from stock 4.1 containing only the
infrastructure needed to select PostgreSQL tests through an environment
variable, including the SQL-owning infrastructure and worker packages beyond
`./domain/...` identified by N16. Do not include pending upstream SQL/schema
fixes or PoC controller runtime support.

The eventual complete PR job must run in under 30 minutes, preferably under
15, including database setup, compilation and cleanup on documented hardware.
Smoke timings do not establish that gate. The following design remains
proposed, not approved; settle it before implementation:

- Use an explicit test-backend selector, provisionally
  `JUJU_TEST_DATABASE=sqlite|dqlite|postgresql`. Keep
  `JUJU_POSTGRESQL_TEST_URL` for connection configuration, rather than
  making the URL implicitly select the backend. Preserve existing
  build-tag-selected defaults when no test backend is requested; reject
  unsupported or inconsistent selections instead of falling back.
- Retain the existing SQLite/dqlite build-tag distinction. Run backend
  permutations as separate test processes or CI jobs, not three copies
  of every suite in one binary. A shared runner can choose the necessary
  build tags; direct `go test` must verify that the requested backend is
  available. Expose actual backend identity separately from SQL dialect:
  SQLite and dqlite share SQL but are not the same backend.
- Select the driver and generated schema together in the test harness.
  Keep environment handling out of production schema constructors.
  Generate PostgreSQL DDL from this branch's 4.1 schema, not copied PoC
  output; do not require moving the existing SQLite schema directories.
- Reuse the PoC's useful driver, error, schema and harness components
  selectively. Retain per-test namespace isolation, bounded connections
  and deterministic cleanup. Do not carry controller configuration,
  bootstrap or PostgreSQL runtime worker support into this branch.
- Keep CI provisioning and PR wiring separately reviewable, potentially
  on a second branch. CI owns a disposable PostgreSQL instance/database,
  supplies its connection configuration and destroys it after the job.
  Evaluate a pinned PostgreSQL service/container and explicit disposable
  server settings before importing the charm-based PoC provisioning.
- Build any demonstration branch separately from 4.1 plus infrastructure
  plus the selected pending upstream branches. Rebuild that branch as
  patches merge; its green result must not become a dependency of the
  independently reviewable infrastructure branch.

Historical timing is promising but does not establish CI readiness: the
PoC's 138-package, warm-cache, untagged runs at `-p 8` took 566-612 s
and still had failures. The unit-test server disabled durability and used
`max_locks_per_transaction=512`. The longest package, rather than the
number of databases, bounded wall time; the database-pool experiment
showed no useful gain. See [Stage 6 measurements](../../stages/06-compatibility.md#database-pool-poc).
Measure the final coverage, race configuration, skips and cold/warm build
costs before choosing the PR gate.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

## Deferred: PostgreSQL test-suite upstreaming

This is not a stock-upstream breakout or a dispatchable standalone fix.
User decision on 2026-11-10: carry the local changestream shutdown
cancellation fix with or after the future effort to run the upstream test
suite on PostgreSQL, rather than factoring it out now.

PoC commit `7e513ceaa8` in `internal/changestream/stream/stream.go` checks
`errors.Is(err, context.Canceled)` without first applying `errors.Cause`.
Its regression covers direct and wrapped cancellation during normal
shutdown. PostgreSQL produced the real failure; natural SQLite/dqlite
reproduction has not been established. Preserve that distinction when
scoping the future PostgreSQL test-suite work. Finding details and local
validation are in UPSTREAM-FIXES.md under stream shutdown.
