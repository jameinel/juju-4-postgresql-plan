# sqlair placeholders for non-SQLite drivers

[Shared direction](../AGENTS.md) | [Upstream dashboard](../UPSTREAM-PLAN.md)

Shared evidence and design comparison for two alternative
[canonical/sqlair](https://github.com/canonical/sqlair) changes. As of
2026-11-14, the first is the primary PR and the second is a linked
comparison branch:

- [`sqlair-dollar-placeholders`](../upstream/sqlair-dollar-placeholders/status.md):
  always emit `$N`, numbered in order of first appearance. No API change.
- [`sqlair-placeholder-style`](../upstream/sqlair-placeholder-style/status.md):
  a per-`DB` option with three styles: `@name` (default), `$N` and
  `:name`. The `?` style was dropped; see that record.

Related, independent of both: [Juju empty-collection membership
(X2)](../upstream/sqlair-empty-in/status.md). Retargeted on 2026-11-14 to
Juju caller fixes, not sqlair slice rendering; scheduled as H3.1 before
affected PostgreSQL H4 acceptance. It does not gate either placeholder
proposal or require another sqlair release.

Goal: sqlair works with pgx (and other non-SQLite drivers) without SQL
rewriting, so the Juju PoC driver no longer rewrites sqlair output (D5;
DIALECT.md X1). Upstream Juju remains SQLite/dqlite-only (D55); these
change only sqlair. Evidence gathered 2026-11-14 against sqlair `main` at
`bd54c49`, unless stated otherwise.

## Findings

### sqlair

- No placeholder abstraction exists. `internal/expr/querybuilder.go`
  hard-codes `"@sqlair_" + N` in `sqlBuilder.writeInputs` and
  `boundInsertColumn.parameter`, and passes `sql.Named("sqlair_N", v)` in
  `addInputs` and `parameter`. Output columns use a separate alias,
  `AS _sqlair_N` (`markerName`), decoded when scanning; it is lower case,
  so PostgreSQL identifier folding does not affect it.
- Input numbers are assigned at bind time (`inputAssigner`). Bulk insert
  assigns a block per column, so the SQL text is not in numeric order:
  `(@sqlair_0, @sqlair_2), (@sqlair_1, @sqlair_3)`.
- Arguments are already appended in first-appearance order (`addInputs`;
  `addInsert` iterates rows, then columns). A single-value insert column
  in a bulk insert reuses its placeholder in every row and appends the
  value once (`parameter`, `newParam`).
- SQL is generated per call in `DB.Query` / `TX.Query`
  (`s.te.BindInputs(...)`, `sqlair.go`); `TX` holds its `*DB`. The
  statement cache keys driver statements on (Statement, DB) and compares
  the primed SQL text (`cache.go` `lookupStmt`), so a per-DB setting needs
  no cache change.
- sqlair has no backend plugins and no driver dependency at runtime. The
  tutorial says it "can be used with any database/driver that works with
  the database/sql package". `NewDB(*sql.DB)` takes no options; the only
  options precedent is the `TXOptions` struct passed to `Begin`.
- The input-expression docs describe sqlair expressions as replacing
  placeholders "such as `?` or `$1`". No docs mention `@sqlair_N`.
- Tests: 44 `@sqlair_` expectations, all in `internal/expr/expr_test.go`.
  Integration tests use mattn/go-sqlite3 (a test-only requirement in
  `go.mod`). CI (`.github/workflows/go-test.yml`) runs `go test -v ./...`
  on Go 1.24.x and 1.x.
- [sqlair#189](https://github.com/canonical/sqlair/issues/189) (open,
  2025-08): a pgx user hits `column "sqlair_0" does not exist`.
  Joe Phillips (manadart) explained the cause; Simon Richardson suggested
  always emitting `$N`, since SQLite accepts `$AAAA`.

### Parser probe (`sqlair.Prepare`)

- Accepted: `name::text`, `tags @> $P.name`, `j ? 'k'`.
- Rejected: `$$it's $P.id$$` (dollar quote parsed as an input
  expression); `E'it\'s $P.id'` ("missing closing quote").
- Silently accepted: `id = $1`, passed through as bypass SQL; it would
  collide with generated `$N`.

Decision (2026-11-14): reject bypass `$<digits>` at `Prepare`. Do not lex
dollar-quoted or `E''` strings; possibly reject them later as non-portable,
after an audit of Juju's sqlair queries confirms that escaping happens
through bound parameters, not literals.

### Drivers and `database/sql`

- Go `database/sql` (go1.27.4) defines no placeholder syntax; query text
  goes to the driver unchanged. `driver.NamedValue.Ordinal` is always set;
  `Name` is optional, has no sigil, and "should" take precedence when set.
  Names must begin with a letter (`validateNamedValueName`), so `$1`
  cannot be bound by name. Drivers without the context interfaces reject
  names ("driver does not support the use of Named Parameters"). Ordinal
  binding is the only contract every driver honors.
- pgx `stdlib` v5.11.0 ignores names and binds by ordinal
  (`convertNamedArguments`, `stdlib/sql.go:911`).
- go-dqlite v1.21.0 sends only ordinals and panics if they are out of
  sequence (`internal/protocol/message.go:164`). On Juju's dqlite,
  `@sqlair_N` names are unused; binding works only because SQLite numbers
  `@sqlair_N` by first appearance and sqlair appends arguments in that
  order.
- SQLite binds `$N` by first appearance, not by the digit. Tested with
  mattn/go-sqlite3 and args `("first", "second")`: `SELECT $2, $1` returns
  `first, second`; `SELECT $1, $2, $1` returns `first, second, first`.
  PostgreSQL binds by the digit. `$N` is portable only when numbers follow
  first appearance.

| Form | SQLite (mattn) | dqlite | PostgreSQL / CockroachDB | MySQL | SQL Server | Oracle |
|---|---|---|---|---|---|---|
| `$N` | ✓ by appearance | by appearance* | ✓ by number | ✗ | ✗ | ✗ |
| `?` | ✓ | ✓ | ✗ | ✓ | ✗ | ✗ |
| `@name` | ✓ | by appearance | ✗ | ✗ | `@name` | ✗ |
| `:name` | ✓ | by appearance | ✗ | ✗ | ✗ | `:name` |
| other | | | | | `@pN` | `:N` |

✓ = verified in source or by test. \* Same SQLite engine; not run on real
dqlite. MySQL, SQL Server and Oracle columns are from memory and
unverified.

## Comparison

| | `sqlair-dollar-placeholders` | `sqlair-placeholder-style` |
|---|---|---|
| API | None | `NewDBWithOptions` with `DBOptions.PlaceholderStyle` |
| Default output | Changes to `$N` | Unchanged (`@sqlair_N`) |
| Drivers tested | SQLite, PostgreSQL | SQLite (all styles), PostgreSQL (`$N`) |
| Regression | SQL Server via go-mssqldb (`@name`/`@pN` only) | None by default |
| Size of own commit | +122/-64, mostly test renumbering | +268/-6: API, style plumbing, per-style tests and docs |

Both share three commits: first-appearance numbering, `$<digits>`
rejection, and the separate `tests/postgresql` module.

## Where decisions live

sqlair has no decision log. The rationale for the placeholder choice is in
the [primary PR description](../upstream/sqlair-dollar-placeholders/pr.md)
and the [#189 comment draft](../upstream/sqlair-dollar-placeholders/reply-issue-189.md).
A decision log is an independent future proposal; see
[`sqlair-decision-records`](../upstream/sqlair-decision-records/status.md).
This planning record holds the comparison until the choice is made, then
links to the sqlair artifacts.

## Shared requirements

- PostgreSQL integration tests live in the separate `tests/postgresql`
  module, gated on `SQLAIR_POSTGRESQL_TEST_URL`, with a `postgres:16`
  service job in `.github/workflows/go-test.yml` (decided 2026-11-14,
  replacing the earlier test-only pgx import in the root module). The root
  module keeps Go 1.24 and no pgx dependency.
- Verify `$N` on real dqlite before proposing: build Juju against the
  sqlair branch (`replace` directive) on a host with `dqlite.h`, or use
  Juju CI, and run a representative set of domain state tests including
  bulk inserts.
- sqlair is not a Juju repository: use sqlair's commands and guidance.
  Follow sqlair's CONTRIBUTING.md (conventional commits). Branches are
  published on the `jameinel/sqlair` fork; each push needs the user's
  approval.
- Juju PoC reconciliation after a sqlair release: use the chosen API on
  the PostgreSQL path; remove `@sqlair_N` rewriting from
  `internal/database/postgresql`, keeping `?` rewriting for raw
  `database/sql` callers; update D5, DIALECT.md X1 and
  `stages/03-driver.md`.

## API audit

Applies to `sqlair-placeholder-style`. User preferences (2026-11-14):
avoid making sqlair aware of the backend; syntax-named styles; type name
`PlaceholderStyle`; value names leaning `AtNamed`, `Dollar`, `Question`,
`Colon` (`At` alone is unclear); `WithPlaceholderStyle(DollarNumbered)`
felt unwieldy, `WithPlaceholderDollarNumbered` was suggested.

Industry audit (links not yet verified, except sqlx at `41dac16`):

- Syntax-named: sqlx
  ([`bind.go`](https://github.com/jmoiron/sqlx/blob/master/bind.go):
  `QUESTION`, `DOLLAR`, `NAMED`, `AT`, mapped from driver names; `NewDb`
  requires a driver name; `Rebind` does not skip literals), squirrel
  ([`placeholder.go`](https://github.com/Masterminds/squirrel/blob/master/placeholder.go):
  `Question`, `Dollar`, `Colon`, `AtP`), Python
  [PEP 249 `paramstyle`](https://peps.python.org/pep-0249/#paramstyle)
  (`qmark`, `numeric`, `named`, `format`, `pyformat`). PEP 249 differs in
  direction: the driver declares the syntax that applications write, and
  may translate it for the database (psycopg2 `%s`). The sqlair analogue is
  SQLAlchemy, which generates SQL in the driver's `paramstyle`. sqlair's
  output is likewise the input of a Go `database/sql` driver, not the
  database protocol. Most Go drivers pass placeholders through, which hides
  the distinction; the Juju PoC wrapper driver, which accepts `@sqlair_N`
  for PostgreSQL, shows it. The setting is therefore a driver property, not
  a database property, which favors syntax names documented as "the
  placeholder syntax your `database/sql` driver accepts".
- Database-named: [bun](https://github.com/uptrace/bun/tree/master/dialect),
  [GORM](https://github.com/go-gorm/gorm/blob/master/interfaces.go)
  (`Dialector.BindVarTo`),
  [goqu](https://github.com/doug-martin/goqu/blob/master/sqlgen/sql_dialect_options.go)
  (database-named registration over property fields),
  [Ent](https://github.com/ent/ent/tree/master/dialect),
  [go-jet](https://github.com/go-jet/jet),
  [sqlc](https://docs.sqlc.dev/en/latest/reference/config.html) (`engine`).
- Hybrid: [SQLAlchemy](https://docs.sqlalchemy.org/en/20/core/internals.html#sqlalchemy.engine.Dialect)
  (database-named dialects, each with a `paramstyle`).

Naming hazards: sqlx `AT` is numbered `@p1`, but sqlair emits named
`@sqlair_N`; PEP 249 `numeric` is `:1`, not `$1`; `Named` alone does not
say which sigil. Candidates (sigil + kind, under a type such as
`PlaceholderStyle`, `BindStyle` or `ParamStyle`):

| Example | Short | Sigil + kind | PEP 249-like |
|---|---|---|---|
| `@sqlair_0` | `At` | `AtNamed` | `AtName` |
| `$1` | `Dollar` | `DollarNumbered` | `DollarNumeric` |
| `?` | `Question` | `QuestionPositional` | `QMark` |
| `:sqlair_0` | `Colon` | `ColonNamed` | `ColonName` |
