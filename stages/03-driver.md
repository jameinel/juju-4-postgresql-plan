# Stage 03: PG driver, placeholder rewriting, errors

Branch `pg-poc/03-driver`, on `pg-poc/02-schema-move`.

- Goal: run sqlair and raw `database/sql` queries against PostgreSQL with
  the same error semantics callers expect.
- Steps:
  1. Add `github.com/jackc/pgx/v5` and a wrapping driver (for example
     `internal/database/postgresql`).
  2. Placeholder rewriter: `@sqlair_N` and `?` to `$N`. It must skip string
     literals, quoted identifiers, comments and dollar-quoted strings, and
     must map a repeated `@name` to a single `$k`.
  3. Extend `internal/database/errors.go` / `drivererrors` with PG SQLSTATE
     mapping (23505, 23503, 23502, 23514; retryable 40001, 40P01, 08xxx).
  4. Teach `txn/transaction.go` retries about PG retryable errors.
  5. Set `default_transaction_isolation=serializable` in the connection
     config (D7). Do not reset the session role; see the Stage 1 findings
     on `login_hook`.
- Verification: unit tests for the rewriter (table driven, including
  literals and repeated names); opt-in integration tests against a real PG.
- Test env var: `JUJU_POSTGRESQL_TEST_URL` (N6).
- Status: in review. Committed on `pg-poc/03-driver` (one `feat(database)`
  commit). Unit and opt-in
  integration tests pass against the Stage 1 PostgreSQL (16.15, plain
  model `pg-poc` on `lxd36-pgtest`) with `-race`.
- Restart notes:
  - Code: `internal/database/postgresql` (`driver.go`, `rewrite.go`, tests),
    `internal/database/drivererrors/postgresql.go` and edits to
    `drivererrors/errors*.go` and `internal/database/errors.go`.
  - API: `postgresql.Open(url, opts...) (*sql.DB, error)` and
    `postgresql.NewConnector(url, opts...) (driver.Connector, error)`;
    option `WithSearchPath(schemas...)`. The driver is also registered
    with `database/sql` as `juju-postgresql`. `sqlair.NewDB(db)` works on
    the result.
  - Run the integration tests with
    `JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url) go test -race ./internal/database/postgresql/`.
    Without the variable the suite skips; the rewriter tests always run.
  - Not run here: `golangci-lint` (deferred while the base lint fix landed),
    and a `-tags dqlite` build (no `dqlite.h` on this host). The
    `errors_linux.go` edits are two one-line changes mirroring
    `errors_other.go`, which is built and tested.
  - Next: user review.
- Findings (for later stages):
  - Rewriting only recognises `@sqlair_<digits>` (not arbitrary `@name`),
    so `@>`, `@@` and other operators pass through. Positional `?` and
    named placeholders may be mixed; `$N` already in the query is left
    alone but must not be mixed with `?`. Queries with no placeholders are
    passed through unchanged, so DDL scripts with several statements work
    (pgx uses the simple protocol when there are no arguments).
  - `search_path` and `default_transaction_isolation` are sent as startup
    parameters. Schema names are double-quoted, so hyphenated names such as
    model UUIDs work. Isolation is forced to `serializable` and cannot be
    overridden by the URL.
  - Role: `current_user` is `charmed_juju_owner` and `session_user` is
    `relation-N` on pooled connections; pgx's `ResetSession` does not run
    `DISCARD ALL` or `RESET ROLE`, so no extra `SET ROLE` is needed.
  - Retry: the transaction runner's `StdTxn`/`Txn` do not retry
    themselves; retries come from `RetryingTxnRunner.Retry`, which uses
    `drivererrors.IsErrRetryable`. That now covers 40001, 40P01, class 08,
    57P0x and pgconn "safe to retry" errors, so no change to
    `txn/transaction.go` was needed. A SERIALIZABLE failure can surface at
    `COMMIT`, which is also covered. The "cannot start a transaction within
    a transaction" `ROLLBACK` workaround there is SQLite only and harmless.
  - Error mapping: PostgreSQL has one SQLSTATE (23505) for unique and
    primary key violations. Primary key is identified by a `_pkey`
    constraint name suffix, which holds for default-named keys; Stage 4
    must not give primary keys custom names. Trigger errors map from
    `P0001` (plain `RAISE EXCEPTION`), so Stage 4's trigger translation
    (D12) should use the default errcode. `ErrConstraintRowID` has no PG
    equivalent and `database.IsErrConstraintRowID` is unchanged, and
    `database.IsExtendedErrorCode` (generic SQLite error) is not extended.
  - `drivererrors.IsError` / `IsErrorTarget` now include `*pgconn.PgError`,
    so `domain` error masking hides PG errors like SQLite ones.
  - `go mod tidy` added `pgx/v5` v5.11.0 plus `pgpassfile`, `pgservicefile`
    and `puddle/v2`.
  - Statement caching: pgx stdlib defaults to cached prepared statements;
    Stage 6 should watch for "cached plan must not change result type"
    errors when DDL changes under a live pool.
