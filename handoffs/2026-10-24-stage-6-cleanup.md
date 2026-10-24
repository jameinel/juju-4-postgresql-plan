# PostgreSQL PoC: handoff, Stage 6 cleanup paused (2026-10-24)

> Point-in-time note. The work stopped because the PoC PostgreSQL server
> ran out of disk. It is being cleaned up or redeployed with more disk in
> another session. Fold the items below into the stage files, then mark
> this note superseded.

Goal of the session: finish Stage 6 (PostgreSQL test harness and suites)
so that Stage 8 can continue.

## 1. What is pushed

The code working tree is clean on `pg-poc/08-dbaccessor`. Every branch
matches `origin` (`jameinel/juju`), and the plan repository is pushed.

| Branch | Tip |
|---|---|
| `pg-poc/00-base-fixes` | `5b1d9dac4e` |
| `pg-poc/01-provision` | `57a97c8b99` |
| `pg-poc/02-schema-move` | `a64da4dc2a` |
| `pg-poc/03-driver` | `d6751cd57e` |
| `pg-poc/04-schemagen` | `7f44bc255b` |
| `pg-poc/05-queries` | `63629d8bd2` |
| `pg-poc/06-test-harness` | `3d26de5819` |
| `pg-poc/07-config` | `34cecbffa6` |
| `pg-poc/08-dbaccessor` | `5f81f2f94f` |

## 2. Done this session

Earlier sessions did the Stage 6 restack, the shared test-schema helper
(`internal/database/postgresql/testing`, `pgtesting`) and the lease
pinning fix (now `fix(lease): resolve lease pin conflicts in sql` on
Stage 5).

This session, in commit order:

- **Stage 3**, `fix(database): rewrite empty in lists for postgresql`.
  sqlair renders an empty slice as `IN ()`, which PostgreSQL rejects. The
  driver rewrites `IN ()` to `= ANY ('{}')` and `NOT IN ()` to
  `<> ALL ('{}')` (DIALECT.md X2). The user approved this as an interim
  workaround. Long term, the query layer should emit dialect-correct SQL.
  It has rewrite unit tests and a sqlair test on PostgreSQL covering int,
  text and NULL operands.
- **Stage 5**, `fix(domain): qualify on conflict upserts for postgresql`
  (portability). It names the conflict targets, qualifies target-table
  columns in `DO UPDATE` (X3, X4) and replaces scalar `MAX(a, b)` with
  `CASE` (X5). Sites: sequence, access user auth, model config, model
  defaults, remote relation status, relation status, status model state
  and modelmigration log transfer.
- **Stage 6**, a fixup squashed into the harness commit. The harness caps
  each PostgreSQL pool at 3 open and 3 idle connections, matching
  `dbaccessor` `applyDBLimits` (X22). Without the cap, parallel tests
  exhausted `max_connections`.
- **Plan**: new [DIALECT.md](../DIALECT.md), a catalogue of SQLite /
  PostgreSQL differences, X1 to X23. PLAN.md session protocol step 6 now
  requires recording differences there. UPSTREAM-FIXES.md gained U7
  (`v_model_config` UNION order), and the U4 / PinLease commit
  references were corrected.

Verified after the restack: the final tree is identical to the
pre-restack tip. Stage 3 driver tests pass on both backends. The Stage 5
affected packages pass on SQLite. On PostgreSQL, `domain/sequence/state`
and `domain/lease/state` pass at Stage 6. `go build ./...` passes at the
Stage 8 tip.

On PostgreSQL, `agent/agentbootstrap` `TestPostgresBootstrapSuite` now
passes; it failed at bootstrap operations 4 and 5 before.

## 3. PostgreSQL server: disk full

- The PoC server is LXD container `juju-404012-0` (10.158.110.68), from
  the Stage 1 script. Its 4.9 GB root filesystem is 100% used, and
  PostgreSQL refuses connections.
- `var/log/postgresql` holds about 1.8 GB. pgaudit plus
  `log_statement = 'ddl'` log the full schema DDL for every test schema,
  and single log files reach 600 to 900 MB.
- `data/logs/16/main` holds about 1.9 GB of WAL: `wal_keep_size = 4096`
  MB, `archive_mode` on with `archive_command = '/bin/true'`.
- The trigger was running four large suites in parallel (application,
  machine, model controller and model state). This broke the "one package
  at a time" rule in PLAN.md.
- For the redeploy, consider:
  - More root disk.
  - Turning off the charm's audit logging (check the charm's option name).
  - A smaller `wal_keep_size`.

  The provisioning script, `scripts/postgresql-poc/deploy-postgresql.py`
  on Stage 1, may need options for these. Update `~/.config/juju-pg-poc/`
  `db-url` (and `db-url-tls`) if the address or credentials change.
- Interrupted runs skip cleanup, so `juju-test-*` schemas leak. The
  `pgtesting` sweep drops schemas whose PID is gone the next time a test
  process opens the admin connection. On a reused server, check this
  with `SELECT nspname FROM pg_namespace WHERE nspname LIKE 'juju-test-%'`.

## 4. Failures seen on PostgreSQL, to rerun

Logs are in the Copilot session folder (`files/phase-e/`,
`files/phased/`). Everything after the disk filled (SQLSTATE 53100, then
57P03) is invalid, so rerun each package on its own after the redeploy.

| Package | Real errors seen | DIALECT / U |
|---|---|---|
| `domain/controllernode` `TestWatcherSuite` (5 tests) | Root cause found, see section 5, item 1 | X16 |
| `internal/database/txn` `TestTxnLogging` | Logs only `COMMIT`. `BEGIN` and query lines come from go-dqlite's context tracer (`internal/database/tracing`), which the pgx driver does not call. The test already skips on SQLite. | Ask the user (section 5, item 2) |
| `agent/agentbootstrap` `TestFreshStateSuite` on PostgreSQL | `fresh_state_test.go` uses bare `user`, which reads `current_user`: count 1, and `uuid = text` | X8 |
| `domain/access/state` (62) | Double-quoted strings (42703), bare `user` (42601), invalid UTF-8 (22021) | X6, X8, X11 |
| `domain/modeldefaults/state` (28) | Double-quoted strings (42703) | X6 |
| `domain/modelconfig/state` (1) | `TestModelConfigAgentStreamWithValueInConfig` | U7, X15 |
| `domain/modelmigration/state/model` (2) | Double-quoted machine UUID; decoded time in `time.Local` | X6, X14 |
| `domain/model/state/controller` (64, partly disk) | `datetime(...)` (10), `iif` (1), bare `user` in test SQL | X7, X8 |
| `domain/model/state/model` (6) | `model table is immutable, only insertions are allowed` (P0001), not yet diagnosed | ? |
| `domain/machine/state` (127, mostly disk) | `column "arm64" does not exist` | X6 |
| `domain/application/state` (608, mostly disk) | `syntax error at or near "ep_names"`, not yet diagnosed | ? |

Fourteen test files still use the table name `user` unquoted (`grep -rlE
'\b(FROM|JOIN|INTO|UPDATE)\s+user\b' --include='*_test.go'`).

## 5. Next steps

Ask the user before 1, 2, 4 and 5; they involve design choices.

1. **Change stream query (X16).** `internal/changestream/stream/stream.go`
   `selectQuery` selects bare `c.edit_type_id`, `created_at` and
   `ORDER BY c.id` with `GROUP BY c.namespace_id, c.changed`. SQLite takes
   bare columns from the row that holds `MAX(c.id)`; this was verified.
   PostgreSQL fails with 42803, so the stream worker dies and never
   reports idle.

   A portable fix: select the `MAX(id)` per group in a subquery and join
   back to `change_log` for the other columns. Ask which stage owns this:
   Stage 5 (shared query) or Stage 6. It is portability, not a dqlite
   bug. Re-run `domain/controllernode` and `domain/changestream`
   afterwards.
2. **`TestTxnLogging`.** Choose between skipping on PostgreSQL (like
   SQLite) and having the pgx wrapper driver emit tracer spans.
3. **Mechanical test-SQL fixes**, as `latent (test)` commits per
   UPSTREAM-FIXES conventions:
   - Quote `"user"` (X8).
   - Use single-quoted literals (X6, U6).
   - Replace `datetime` / `iif` in test SQL (X7).
   - Insert U5 upgrade node ids as text.
4. **X14 time location.** Fix it in the driver (UTC decoding), or have
   tests compare with `.UTC()`.
5. **X11 invalid UTF-8**, and **U7 `v_model_config`**. U7 needs its own
   `bug` commit.
6. **Undiagnosed errors.** Investigate `ep_names` (application) and the
   immutable model trigger (model/state/model).
7. **`RAISE(ABORT, ...)` conversions.** Convert the four remaining test
   sites to `schematesting.FailingTriggerSQL`: removal relation tests,
   `application_refresh_test.go` and `reprovision_test.go`.
8. Re-run the Stage 6 suites one package at a time on both backends:
   - `domain/model/state/controller` and `domain/model/state/model`
   - `domain/machine/state` (about 5 minutes)
   - `domain/application/state` (about 17 minutes)

   Use `-race` where they are verified.

## 6. Documentation still owed

Phase F of this cleanup:

- **DECISIONS**, both confirmed with the user and not yet written:
  - **N9**, the test-schema convention:
    - `juju-test-<pid>-<n>[-<suffix>]` names.
    - A `COMMENT ON SCHEMA` with pid, start time and test name.
    - A once-per-process sweep of schemas whose PID is gone.
    - Quoting with `pgx.Identifier`.
  - **N10**, backend-only skips: `SkipOnPostgreSQL` with a "dqlite only:
    <reason>" message, called before `SetUpTest`. Parameterised
    two-backend tests later.
- **Stage files**:
  - 03: helper, empty `IN` rewrite.
  - 04: no change expected.
  - 05: ON CONFLICT / MAX commit, PinLease.
  - 06: commit table is stale since the restack; pool cap; findings
    above.
  - 08: bootstrap now passes on PostgreSQL.
- **PLAN.md**: status table and "Next" line.
- **DIALECT.md**: keep current. X16 now records the SQLite MIN/MAX
  bare-column rule.
