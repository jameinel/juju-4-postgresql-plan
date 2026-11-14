# sqlair-placeholder-style

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Claimed 2026-11-14 by session `6e9894b2-d722-418e-9dac-32096188585b`.
Handoff updated 2026-11-14; GitHub outcomes below were freshly checked
against the published heads, not inferred from earlier runs.
The user approved splitting infrastructure first, then interpolation.
Both branches are published on `jameinel/sqlair`. The latest approved push
updated them atomically, using the original remote SHA as a rewrite lease
for the rebased interpolation branch:

| Layer | Branch/worktree under `canonical/sqlair/.worktrees/` | Parent | Head |
|---|---|---|---|
| Standalone PostgreSQL test infrastructure | `postgresql-test-infrastructure` | `bd54c49` (`origin/main` at original creation) | `ccf6072c7527d0d897c26d4c7063c687cd0d92b6` |
| Interpolation and its PostgreSQL driver test | `placeholder-styles` | `ccf6072` | `69db9bcf1ff0f7d37d0044450a375a36be28adc6` |

The following independent follow-ups are now published with explicit
approval:

- `7e6a03d build: require Go 1.26`: minimum Go 1.26.0, CI matrix
  `1.26.x`/latest, `GOTOOLCHAIN=local` to prevent silent compiler upgrades,
  and matching contributor/tutorial requirements. Go 1.26 matches the
  default compiler in Ubuntu 26.04 LTS and remains upstream-supported.
- `10eaa2e fix: update PostgreSQL test dependencies`: pgx v5.13.0,
  x/text v0.44.0, tidy transitive dependencies, and contributor guidance
  for required local checks.
- `ccf6072 docs: add repository agent instructions`: root `AGENTS.md`,
  separate from baseline and dependency changes, documenting architecture,
  validation requirements, PostgreSQL isolation and cleanup.

Infrastructure head is `ccf6072c7527d0d897c26d4c7063c687cd0d92b6`.
The interpolation commit was rebased onto it, producing
`69db9bcf1ff0f7d37d0044450a375a36be28adc6`; `git range-diff` confirms its
patch is unchanged from `1f7154d`.
The atomic push fast-forwarded infrastructure and rewrote interpolation
with `--force-with-lease` pinned to its original remote tip
`1f7154d0ef58f264d9e3c43be21dbe44239e9eee`.
Both remote heads were verified afterward. The original source branch,
`main`, and the other session's dollar-only branch were not changed.
No PRs, comments, or stack metadata were published.

GitHub Actions outcomes for these exact heads (2026-11-14):

| Branch/head | Go Test (`1.26.x` and latest stable) | govulncheck | Documentation |
|---|---|---|---|
| Infrastructure `ccf6072` | [Passed](https://github.com/jameinel/sqlair/actions/runs/43624704961) | [Passed](https://github.com/jameinel/sqlair/actions/runs/43624704957) | [Failed: spelling](https://github.com/jameinel/sqlair/actions/runs/43624705538) |
| Interpolation `69db9bc` | [Passed](https://github.com/jameinel/sqlair/actions/runs/43624704948) | [Passed](https://github.com/jameinel/sqlair/actions/runs/43624704940) | [Failed: spelling](https://github.com/jameinel/sqlair/actions/runs/43624705430) |

Both documentation runs build HTML successfully and pass link and
inclusive-language checks. Both flag existing security-documentation
terms: `Cryptographic`/`cryptographic`, `TLS`, and `utilizing`.
Interpolation additionally flags `pgx` and `afterward` in the newly added
database how-to text. The latter findings belong to this change; they are
not pre-existing failures. Overall CI is not green.

Remaining work:

- Correct the documentation spelling failures, distinguishing the new
  how-to findings from existing security-documentation findings.
- Validate the `$N` style on real dqlite; SQLite and PostgreSQL evidence
  does not replace this supported-backend check.
- Coordinate the shared decision-log/backfill commits and placeholder ADR
  with the dollar-only alternative; their wording remains unconfirmed.
- Prepare local PR descriptions for the two layers and obtain approval
  before publication. This session has created no PRs and performed no
  upstream landing or PoC reintegration.

The original branch/worktree `placeholder-style` and its published head
`cfafcde23110d6cff4c6978d1f88acdd25c65209` remain unchanged as the source
snapshot; use the two new branches for further work.

Target repository:
[canonical/sqlair](https://github.com/canonical/sqlair), base `main` at
`bd54c49` when planned. Alternative to
[`sqlair-dollar-placeholders`](../sqlair-dollar-placeholders/status.md);
both are prepared and proposed together, and the maintainers choose one.

The interpolation implementation adds `NewDBWithOptions` and `DBOptions`, with
`PlaceholderStyle` values `AtNamed`, `Dollar`, `Question`, and `Colon`.
The user confirmed the options-struct API and these names on 2026-11-14.
`NewDB` is unchanged; options are copied and transactions inherit the style.
Unsupported style values produce query errors.

All four styles pass the existing SQLite package suite. Focused expression
tests cover slices, bulk inserts with two bulk columns and a reused scalar,
omitted columns, literal values, first-appearance numbering and argument
forms. Raw `$N` fails at prepare time; raw `?` fails at bind time only in
`Question`, with quoted strings, identifiers and comments excluded.

The shared driver helper covers bulk inserts, slices, an uncached
transaction (rolled back), database statement preparation and cached
transaction execution. The PostgreSQL test is gated on
`SQLAIR_POSTGRESQL_TEST_URL`, uses pgx,
creates a unique schema and drops it afterward. CI provisions PostgreSQL
16. The helper passes against the existing PostgreSQL 16 instance.
No database or local PostgreSQL server was created.

The standalone infrastructure branch contains the fixture, cleanup
command, native-pgx tests, dependencies, contributor documentation and CI,
plus the separately committed Go baseline and repository agent guidance.
It does not change `sqlair.go`, `internal/expr`, or `package_test.go`.
The upper branch adds interpolation, SQLite style coverage, user
documentation and `postgresql_test.go`.

The user rejected the original fixed CI password. On explicit approval,
the infrastructure workflow now generates 32 random password bytes as hex
per matrix job, masks the password and URL, starts PostgreSQL 16 from a
workflow step on a dynamically assigned loopback port, and removes the
container in an `always()` step. It does not repurpose the GitHub API token.
Before the separate baseline, dependency and agent-guidance follow-ups,
the split upper tree matched the original source apart from this workflow
change and the corresponding contributor documentation. The interpolation
patch itself remains unchanged after rebasing.

Independent validation for both new branches: PostgreSQL-backed
`go test -timeout 2m -p 4 -parallel 4 ./...`, an unset-URL
`go test -timeout 2m ./...`, and `go vet ./...` pass.
Workflow YAML parses and every shell step passes ShellCheck.
Infrastructure GitHub Actions run `43623091542` and interpolation run
`43623249772` passed both Go matrix jobs, including generated-password
PostgreSQL startup, tests and cleanup.

Earlier published snapshots failed vulnerability checks. Reproduced locally
with `go run golang.org/x/vuln/cmd/govulncheck@latest ./...`:
pgx v5.7.6 has GO-2026-5004 (fixed in v5.9.2), and x/text v0.24.0 has
GO-2026-5970 and GO-2026-6629 (both fixed by v0.41.0). Both minimum fixed
versions declare Go 1.25.0, but that release line is no longer supported.
The user approved moving the baseline in a separate commit and preferred
supported releases aligned with Ubuntu LTS; Go 1.26 was selected.
Local follow-ups now pass govulncheck with no vulnerabilities on both
Go 1.26.11 and 1.27.4, for both branches.
Both local branches also pass tests with and without PostgreSQL and
`go vet ./...` on both toolchains. Fixture/cleanup race tests pass on
Go 1.26.11. Formatting and `go mod tidy -diff` are clean; the CI YAML parses
with the expected compiler matrix and disabled automatic switching.
There is no configured golangci-lint check. Contributor and agent
instructions now cover the required checks.

The current documentation failures and their exact-head evidence are
listed in the handoff above, including the new interpolation how-to terms.
No PRs, comments, or stack metadata have been published. The Stacks API
probe succeeded, but native stacks cannot register cross-fork PR heads.

On 2026-11-14 the user requested general PostgreSQL test isolation and
stale-artifact cleanup, and confirmed a Go implementation with a 24-hour
default dry run, explicit `--apply`, active-schema protection, four parallel
fixtures and two pooled query connections per fixture.

`internal/pgtest` now provides `New(t)` and `Open(ctx, url)`: unique marked
schemas, per-connection `search_path` without `public`, normal pool/schema
cleanup, and advisory locks for active-schema protection and a database-wide
four-fixture budget across test processes. Control connections are additional
to the eight-query-connection limit; waiting processes can also hold control
connections. Schema creation and its ownership marker commit atomically.
The PostgreSQL driver test uses this fixture.

`go run ./cmd/sqlair-pg-clean` reports eligible old schemas without
deleting them. `--older-than 48h --apply` explicitly removes old, unlocked
artifacts owned by the current role, with an exact-name `--schema` filter
available. Names, markers and schema identities are checked before deletion.
`CONTRIBUTING.md` documents both the fixture and the command.

Infrastructure tests cover parallel same-table-name isolation, both pooled
connections' search paths and the two-connection cap, database-wide slot
contention, fixture cleanup, failed-open resource release, dry runs, age and
marker checks, active locks, and cleanup after a lock-holder disconnect.
All deletion tests restrict themselves to explicitly created test schemas.
The cleanup command's real-server dry run reported no eligible artifacts.

Historical combined-branch commits created at the user's request on
2026-11-14 (superseded by the split above):

- `119d413 feat: add per-database placeholder styles`: API, binding, parser
  collision checks, SQLite tests for all styles and user documentation.
- `d2a4f08 test: add isolated PostgreSQL fixtures and artifact cleanup`:
  pgx dependencies, isolated fixtures, cleanup command, PostgreSQL driver
  entry point, CI service and contributor documentation.
- `cfafcde test(pgtest): report leftover objects before cascade cleanup`:
  testing-infrastructure-only teardown policy, explicit table cleanup,
  regression test and contributor documentation.

Committed follow-up approved on 2026-11-14: normal fixture cleanup first
tries `DROP SCHEMA` without `CASCADE`. If leftover objects cause PostgreSQL
error `2BP01`, it preserves that error and attempts `CASCADE` as a cleanup
backstop. Other failures are reported without a cascade retry. Tests must
explicitly clean up their objects, following the existing SQLair
`dropTables` pattern. The parallel-isolation test now drops its table, and
a regression test verifies that leftover objects produce an error even
when the backstop successfully removes the schema. The stale-artifact
command retains its direct `CASCADE` policy. PostgreSQL-backed
`go test -timeout 2m -p 4 -parallel 4 ./internal/pgtest .`,
`go vet ./internal/pgtest`, and `git diff --check` pass for this follow-up.

The first commit was independently checked in a temporary detached worktree
with `go test -timeout 2m ./...` and `go vet ./...`, without the PostgreSQL
infrastructure. The combined head passes PostgreSQL-backed package testing
with `go test -timeout 2m -p 4 -parallel 4 ./...`.

Validation at this state: `go test -timeout 2m ./...` both
with and without the PostgreSQL URL set, `go vet ./...`, and
`git diff --check` pass. PostgreSQL infrastructure and command tests also
pass with:

```bash
go test -race -timeout 2m -parallel 4 ./internal/pgtest ./cmd/sqlair-pg-clean
```

Real-dqlite validation remains pending. Documentation HTML builds pass in
CI; a local documentation build was not run because no Sphinx executable
was available. The shared decision-log/backfill commits and the
placeholder ADR are not implemented.

Published with explicit user approval on 2026-11-14: branch
`jameinel/sqlair:placeholder-style` at
`cfafcde23110d6cff4c6978d1f88acdd25c65209`. The remote head was verified
against the local head after a non-forced push. No PR or comment was created.
The history still has the interpolation commit followed by two testing
infrastructure commits.

The approved split is complete locally and published in dependency order.
The standalone infrastructure commit can be reused by the dollar-only
alternative without taking any interpolation API changes.

The dollar-placeholder session has separately claimed its alternative.
The raw-placeholder parser check is included in the interpolation commit
`69db9bc` (previously `1f7154d`); coordinate extraction for the dollar-only
alternative rather than assuming that full commit is style-independent. The PostgreSQL
fixture is now independently reusable in `9efe316`.
`driver_test.go` contains SQLite statement-leak instrumentation rather
than backend-neutral query cases, so the shared functional helper lives
in `package_test.go`; the PostgreSQL entry point is `postgresql_test.go`.

### Comparison with the Juju PoC harness

Read-only comparison against Juju PoC `f3340c6710`, principally
`internal/database/postgresql/testing/schema.go`, `dialect.go`,
`internal/database/testing/dqlitesuite.go`, and
`scripts/postgresql-poc/drop-juju-schemas.py`:

- Both isolate individual fixtures in separate schemas and configure
  `search_path` when connecting, allowing packages to use the same table
  names on a shared database. SQLair's isolation is per fixture, not an
  automatic namespace allocated once for each package.
- SQLair caps its fixtures across processes at four, with two query
  connections each. Juju's domain harness caps each namespace pool at
  three and permits package parallelism, with per-process schema-change
  advisory locks rather than a database-wide fixture semaphore.
  SQLair's budget does not coordinate with Juju or unrelated clients.
- SQLair uses random schema IDs, creation timestamps and live advisory
  locks, without assuming shared host PIDs. Juju uses PID/counter names,
  process/test comments and an automatic stale-process sweep that assumes
  a common host/PID namespace. SQLair only sweeps when explicitly invoked.
- Juju's standalone tool additionally handles controller/model schemas,
  LXD safety checks and PID reuse. These are deliberately absent from
  SQLair's test-only Go tool.
- The user explicitly declined cross-schema dependency inspection:
  SQLair tests should not create such dependencies. Normal drops without
  `CASCADE`, error reporting and a cascade backstop provide the selected
  cleanup policy. There are still no default server-side
  `lock_timeout`, `statement_timeout`, or
  `idle_in_transaction_session_timeout`. SQLair bounds setup/cleanup with
  contexts but does not apply those defaults to arbitrary test queries.
  Schema/search-path isolation is not a permission boundary; tests must
  not create dependencies on another fixture's schema.

Next actions:

- [x] Before coding, confirm the API shape and names with the user
  (Open decisions).
- [x] Reserve in the dashboard; record owner/session here.
- [x] Create worktree `canonical/sqlair/.worktrees/placeholder-style` on
  branch `placeholder-style` from current `origin/main`; record the base.
- [x] Publish independent infrastructure and interpolation layers, then
  separate baseline, dependency and agent-guidance commits with approval.
- [x] Verify both supported Go lines locally and successful test and
  vulnerability CI for the exact published heads.
- [ ] Correct the documentation spelling failures recorded above.
- [ ] Coordinate shared commits, complete decision records, and validate.
- [ ] Verify the `$N` style on real dqlite (shared requirements).
- [ ] Prepare PR descriptions locally; PR publication and any additional
  branch updates require the user's approval.

## Scope

Add a per-`DB` placeholder style with four variants, keeping today's
output as the default. Fixes
[sqlair#189](https://github.com/canonical/sqlair/issues/189) for drivers
that need `$N`, and makes MySQL (`?`) and Oracle (`:name`) usable. This is
the extensible option.

Out of scope: database-named dialects, driver auto-detection, dollar-quoted
or `E''` lexing, empty `IN ()`
([`sqlair-empty-in`](../sqlair-empty-in/status.md)).

## Design

Styles; the argument list is always built in first-appearance order:

| Style | SQL | Arguments | Reused parameter |
|---|---|---|---|
| `@name` (default) | `@sqlair_<id>` | `sql.Named("sqlair_<id>", v)` | same name; byte-identical to today |
| `:name` | `:sqlair_<id>` | `sql.Named("sqlair_<id>", v)` | same name |
| `$N` | `$<position>` | positional | same `$<position>` |
| `?` | `?` | positional | value appended again |

`$N` positions follow first appearance, so they bind correctly on SQLite
and PostgreSQL (shared findings).

1. Public API: a `PlaceholderStyle` type whose zero value is the `@name`
   style, selected per `DB` on `NewDB`; `TX` uses its `DB`'s style. No
   auto-detection: `*sql.DB` does not expose the driver name.
2. Internals: thread the style from `DB` / `TX` into
   `TypeBoundExpr.BindInputs` and `queryBuilder`. Replace the hard-coded
   `"@sqlair_"` sites with style methods that render a placeholder for a
   parameter (id, first-appearance position) and decide whether a reuse
   appends its value again. Keep the `@name` path producing identical SQL
   and arguments, so existing expectations stay unchanged. `Statement`
   stays style-independent; the statement cache already keys on
   (Statement, DB).
3. Parser: reject bypass `$<digits>` at `Prepare` in every style (shared
   commit). For the `?` style, bypass `?` outside literals and comments
   would shift arguments: have the parser record whether bypass SQL
   contains `?`, and fail at bind time in the `?` style with a clear
   error. The `?` stays valid for other styles (PostgreSQL JSON
   operator).

## Confirmed API (2026-11-14)

The user selected the `TXOptions` pattern:
`NewDBWithOptions(db, &DBOptions{PlaceholderStyle: Dollar})`, leaving
`NewDB` unchanged. The constants are `AtNamed`, `Dollar`, `Question`,
and `Colon`. The zero value and nil options preserve `AtNamed`.

### Alternatives considered

- API shape candidates:
  - `NewDB(db, WithPlaceholderStyle(Dollar))` (variadic options);
  - one option per style, for example `WithPlaceholderDollarNumbered()`
    (user suggestion);
  - an options struct, matching the `TXOptions` precedent:
    `NewDBWithOptions(db, &DBOptions{PlaceholderStyle: Dollar})`.
  Making `NewDB` variadic is source-compatible except where `NewDB` is
  used as a function value.
- Value names. User leaning (2026-11-14): `AtNamed`, `Dollar`,
  `Question`, `Colon`. Consider consistency (`ColonNamed`, or
  `DollarNumbered`) and the naming hazards in the shared API audit.

## Commits

Reuse the shared commits from
[`sqlair-dollar-placeholders`](../sqlair-dollar-placeholders/status.md)
(parser rejection; PostgreSQL driver-suite harness and CI). Whichever
session finishes them first records the hashes in both records.

1. `feat(parser): reject positional $N parameters` (shared).
2. `refactor(expr): route placeholders through a style`: internal style
   with only `@name`; no output change; existing tests unchanged.
3. `feat: add placeholder styles`: the public API, the `:name`, `$N` and
   `?` styles, the `?` bypass check, and table-driven `internal/expr`
   cases for each style (bulk insert mixing bulk and single-value
   columns, slices, omitted columns, literals; SQL text and arguments).
4. `test: run the driver suite against PostgreSQL` (shared harness), plus
   running the SQLite driver suite once per style (mattn accepts all
   four); PostgreSQL uses `$N`.
5. `docs: explain placeholder styles`: `docs/howto/db.md` section on
   choosing a style with a driver table, and a new record in
   `docs/decisions/` (next number after the backfill) with an index row.
   The record covers syntax-named styles rather than database dialects;
   why the setting belongs to the driver; the first-appearance
   invariant; why raw `$N` and, in the `?` style, raw `?` are rejected;
   and the always-`$N` alternative it was weighed against. Status:
   Proposed until merged.

The two shared decision-log commits from
[`sqlair-decision-records`](../sqlair-decision-records/status.md) come
before commit 1.

## Validation

- `go test ./...` with no PostgreSQL URL (SQLite suite, all styles).
- `SQLAIR_POSTGRESQL_TEST_URL=... go test ./...` against PostgreSQL 16
  through pgx `stdlib` with the `$N` style.
- Real dqlite through Juju with the default and `$N` styles.
- `go vet ./...`; build the docs if the docs toolchain is available.

## Acceptance criteria

- Default style: generated SQL and arguments byte-identical; existing
  tests pass unchanged.
- All four styles pass the SQLite driver suite; `$N` passes on
  PostgreSQL without SQL rewriting.
- In the `?` style, a reused single-value column in a bulk insert binds
  correctly (value repeated), and bypass `?` fails with a clear error.
- `$1` in bypass SQL fails at `Prepare` with a clear error.
- The PR description links the alternative PR and states that the
  default is unchanged.
