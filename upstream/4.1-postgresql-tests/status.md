# 4.1-postgresql-tests

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Owner/session: `0ec926e6-3858-4f43-bc9c-658024127e88`, assigned by the user
on 2026-11-12. Branch `4.1-postgresql-tests`, worktree
`../4.1-postgresql-tests`; refreshed upstream base
`b36eefc4b34badcdef972c859e96ccb974c592e2`, layout prerequisite
`9da8100e161f59ce5a86485e0725db1b8101154d`, local configuration commit/head
`6e677c28505645413b55862e732c28e576ced3b9`
(`feat(database): centralize test backend selection`). The worktree is clean.
The previously uncommitted configuration work was preserved through a
fast-forward onto layout; no stash, reset or history rewrite was needed.
The user explicitly approved pushing the four exact branch heads on
2026-11-12. They were pushed atomically to `jameinel/juju` and verified with
`git ls-remote`. No PR creation or further publication is authorized.

The approved [upstream-4.1 integration plan](integration-plan.md) owns the
branch graph, staged scopes, exact initial pending-PR inventory and landing
gates. The user approved an independent move-only schema-layout branch,
followed by infrastructure, separately reviewable CI wiring, and a disposable
combined-fix demonstration. The first-class dialect layout is an architectural
prerequisite, not PostgreSQL code hidden in a testing directory.

Local refs `4.1-postgresql-tests-ci` and
`4.1-postgresql-tests-integration` are reserved at the configuration head.
They have no implementation-specific commits or separate worktrees yet.

The user approved `JUJU_TEST_DATABASE` plus `JUJU_TEST_POSTGRESQL_URL`.
Keep environment handling central: a new domain suite must inherit backend
selection through the shared harness, not read these variables itself.
The URL may remain configured while selecting SQLite or dqlite; its
presence must neither select PostgreSQL nor invalidate another backend.

Implemented: shared configuration in `internal/database/testconfig`,
environment preservation, and wiring into `DqliteSuite`. Existing domain
suites inherit selection without modification. `Backend()` exposes actual
SQLite/dqlite identity separately from SQL dialect. Unset/empty selection
retains build-tag defaults; incompatible SQLite/dqlite selections fail.
The PostgreSQL URL is ignored for other backends, even when malformed.

PostgreSQL selection without a URL fails configuration validation; with a
URL it fails explicitly at suite setup because that backend is not
implemented yet. This is a temporary safety gate, not PostgreSQL support.
No upstream SQL fixes, PoC runtime code, dependency changes, or skips were
added. Contributor instructions describe the current limits.

Next actions:

1. Settle the remaining PostgreSQL driver/schema design below, then replace
   the temporary gate with isolated PostgreSQL connections and matching
   schema application. Both must consume central configuration; do not add
   environment handling to individual domain suites.
2. Keep the GitHub provisioning/job branch independently reviewable; its
   [reservation record](../4.1-postgresql-tests-ci/status.md) is not completion.
3. Build a disposable integration branch from stock 4.1, the infrastructure,
   and selected pending upstream fixes. Do not mix those fixes into this
   infrastructure branch. See its
   [reservation record](../4.1-postgresql-tests-integration/status.md).

PostgreSQL execution, the full SQL-owning package selection, and the
complete-job timing gate remain unvalidated.

This is separate test-infrastructure work, not a stock SQL/schema fix;
see the [PoC scope](../../PLAN.md#postgresql-sql-regression-branch).

## Configuration validation (2026-11-12)

Revision: uncommitted configuration patch on base `bd970f693c`. Commands
ran from the branch worktree using existing shared `_deps`; no dependency
installation was needed.

- Unset selector/URL: `go test -race -p 2 -count=1 -timeout 120s
  ./internal/database/testconfig ./internal/database/testing
  ./internal/testhelpers ./domain/sequence/state` passed.
- Explicit dqlite with `JUJU_TEST_POSTGRESQL_URL` set to an intentionally
  invalid connection string: `make run-go-tests
  TEST_PACKAGES='./internal/database/testconfig ./internal/database/testing
  ./internal/testhelpers ./domain/sequence/state'
  TEST_ARGS='-race -p 2 -count=1 -timeout 120s'` passed with actual
  `dqlite,libsqlite3` tags and bundled musl/CGO paths.
- Explicit SQLite with the same unused URL:
  `go test -race -p 2 -count=1 -timeout 60s ./internal/database/testing
  ./domain/sequence/state` passed. The domain package was not modified.
- Configuration tests cover unset/empty selectors, both compiled defaults,
  explicit/unavailable/unknown backends, required PostgreSQL URLs, URL and
  keyword connection representations, switching with the URL retained, and
  preservation across `OsEnvSuite` setup.
- Separate harness invocations confirmed expected nonzero exits for missing
  PostgreSQL URL, not-yet-implemented PostgreSQL backend, and requesting
  dqlite in an untagged SQLite build. The expected diagnostic was checked in
  each output; these are negative checks, not passing PostgreSQL runs.
- Race-enabled SQLite and dqlite harness binaries each passed
  `timeout 35s stress -p 2 -count 20 -timeout 30s -failfast <binary>
  -test.run=TestDatabaseSuite -test.timeout=20s`: 20 runs, zero failures per
  backend. Selector unset and unused PostgreSQL URL configured in both.
- `golangci-lint run -c .golangci.yml ./internal/database/testconfig
  ./internal/database/testing ./internal/testhelpers`: zero issues.
  `txncheck "$PWD"` and `git diff --check` passed. Touched Go files were
  formatted with `gci` and `gofmt`.
- IDE build reported success with limited diagnostics. IDE file inspections
  could not analyse the ignored worktree; actual CLI compilation/tests and
  scoped lint above are the verification evidence.

Logs are in session `0ec926e6-3858-4f43-bc9c-658024127e88`'s `files/`
directory (`testconfig-*.log`). Generated stress binaries were removed.
These are focused configuration checks, not full-suite acceptance or
evidence for the eventual 15/30-minute job target.

## Restacked checkpoint validation (2026-11-12)

The same configuration changes were revalidated on the refreshed layout
parent `9da8100e16` before commit `6e677c2850`:

- Explicit SQLite and real dqlite runs passed for
  `./internal/database/testconfig ./internal/database/testing`
  `./internal/testhelpers ./domain/sequence/state`, with race enabled,
  `-p 2 -count=1 -timeout 90s`, and an intentionally invalid unused
  `JUJU_TEST_POSTGRESQL_URL`. Dqlite used `make run-go-tests`.
- Race-compiled shared-harness binaries each passed 20 stress runs with
  `-p 2 -timeout 30s -failfast`, outer timeout 35 seconds, explicit matching
  selectors and that same unused URL.
- Missing PostgreSQL URL, not-yet-implemented PostgreSQL, and unavailable
  dqlite each failed with the expected diagnostic in separate invocations.
- Scoped `golangci-lint` reported zero issues; `txncheck`, `gci` and
  `git diff --check` passed.

Logs use `restacked-harness-*.log` in this session's `files/` directory.
The infrastructure-only commit changes six files (+339/-1), excluding the
independently committed relocation. No PostgreSQL run or CI job was performed.

<!-- Migrated from PLAN.md; historical evidence retains its revision. -->

#### Infrastructure workflow discussion (2026-11-11)

Local branch `4.1-postgresql-tests` was created at upstream 4.1
`bd970f693c`. Its worktree is
`../4.1-postgresql-tests`, alongside the PoC and planning repositories
under the primary checkout's locally Git-ignored `.worktrees/`. It links
`_deps` to the primary checkout's `../../_deps`; the PoC retains its own
local copy. Future worktrees for this effort also belong under
`.worktrees/`. At that reservation no implementation commits or pending
upstream patches had been added. The PoC's tracked files were unchanged.

The confirmed scope is a branch directly from stock 4.1 containing only the
infrastructure needed to select PostgreSQL tests through an environment
variable, including the SQL-owning infrastructure and worker packages beyond
`./domain/...` identified by N16. Do not include pending upstream SQL/schema
fixes or PoC controller runtime support.

The eventual complete PR job must run in under 30 minutes, preferably under
15, including database setup, compilation and cleanup on documented hardware.
Smoke timings do not establish that gate. The configuration decision below
was confirmed on 2026-11-12; the remaining implementation design is still
proposed and must be settled before its corresponding work:

- Confirmed: use `JUJU_TEST_DATABASE=sqlite|dqlite|postgresql` and
  `JUJU_TEST_POSTGRESQL_URL` for connection configuration. Environment
  handling belongs in the shared test infrastructure. Ignore the URL when
  another backend is selected, and preserve build-tag-selected defaults when
  no backend is requested. Unsupported selections must not fall back.
- Retain the existing SQLite/dqlite build-tag distinction. Run backend
  permutations as separate test processes or CI jobs, not three copies
  of every suite in one binary. A shared runner can choose the necessary
  build tags; direct `go test` must verify that the requested backend is
  available. Expose actual backend identity separately from SQL dialect:
  SQLite and dqlite share SQL but are not the same backend.
- Select the driver and generated schema together in the test harness.
  Keep environment handling out of production schema constructors.
  Generate PostgreSQL DDL from this branch's 4.1 schema, not copied PoC
  output. The user's later 2026-11-12 decision supersedes the earlier no-move
  proposal: first relocate existing assets to `domain/schema/dqlite` on the
  independent `4.1-schema-layout` branch. Add the conversion generator and
  first-class `domain/schema/postgresql` assets only on infrastructure.
- Reuse the PoC's useful driver, error, schema and harness components
  selectively. Retain per-test namespace isolation, bounded connections
  and deterministic cleanup. Do not carry controller configuration,
  bootstrap or PostgreSQL runtime worker support into this branch.
- Keep CI provisioning and PR wiring on the separate
  `4.1-postgresql-tests-ci` branch. CI owns a disposable PostgreSQL
  instance/database, supplies its connection configuration and destroys it
  after the job.
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
