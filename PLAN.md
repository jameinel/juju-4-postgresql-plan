# Juju 4 on PostgreSQL: proof-of-concept plan

Status: **DRAFT, awaiting review**. Background and evidence are in
[ANALYSIS.md](ANALYSIS.md).

This plan spans many working sessions. It is written so any stage can be
picked up cold.

## How to use this document

Each stage runs in its own agent session with a fresh context. Nothing
carries over between sessions except this plan, ANALYSIS.md, the
repository (branches and commits) and the local environment. Each session
must leave in this plan everything the next one needs.

Kick-off prompt for a new session (replace `N`; name the PostgreSQL host
controller only for stages that need a live PostgreSQL, such as 1, 3, 6 and
8 to 10):

```
Read docs/implementation-plans/postgresql/PLAN.md and run Stage N
following its session protocol. Use the <controller> controller.
```

Session protocol:

1. Start on the integration branch, `juju-4-postgresql`, which holds the
   latest plan. Read this plan, the ANALYSIS.md sections the stage cites,
   and the `AGENTS.md` files that apply to the code the stage touches.
2. Work only on the stage named in the prompt. If any stage it depends on
   (see the dependency graph) is not `done`, ask the user before starting.
3. Read the stage's **Status**, **Restart notes** and **Findings**, and the
   findings of the stages it depends on.
4. Confirm the stage's open questions, and any new design choice, with the
   user before writing code. Record decisions in the
   [decision log](#decision-log) only after the user has confirmed them.
5. Create or check out the stage's branch (see [branching](#branching)) and
   commit code only there.
6. Do not start another stage, even if this one finishes early.
7. Before ending the session, update the stage's **Status**, **Restart
   notes** (what is done, what is next, gotchas) and **Findings** for later
   stages, and the [status table](#status). Commit plan changes only on the
   integration branch, then restack as described under
   [branching](#branching).

## Status

| # | Stage | Branch | Status |
|---|---|---|---|
| 0 | Plan documents | `juju-4-postgresql` | in review |
| 00 | Base fixes | `pg-poc/00-base-fixes` | done, in review |
| 1 | PostgreSQL provisioning script | `pg-poc/01-provision` | in review |
| 2 | Move schema to `domain/schema/dqlite` | `pg-poc/02-schema-move` | in review |
| 3 | PG driver, placeholder rewriting, errors | `pg-poc/03-driver` | in review |
| 4 | PG schema generation | `pg-poc/04-schemagen` | done, in review |
| 5 | Query portability fixes | `pg-poc/05-queries` | in review |
| 6 | PG test harness for domain state tests | `pg-poc/06-test-harness` | not started |
| 7 | `db-backend` / `db-url` config plumbing | `pg-poc/07-config` | in review |
| 8 | PG dbaccessor worker and bootstrap | `pg-poc/08-dbaccessor` | not started |
| 9 | LXD bootstrap and smoke test | `pg-poc/09-bootstrap` | not started |
| 10 | enable-ha | `pg-poc/10-ha` | not started |
| 11 | Future work | n/a | not planned |

Dependency graph:

```
00 ──> 02 ──> 03 ──┬──> 04 ──┐
                   └──> 05 ──┴──> 06 ──> 07 ──> 08 ──> 09 ──> 10

00 ──> 01 (standalone)
```

The stages form one stack (N8), with at most two in flight at once. Each
stage depends on every stage below it.

- 04 and 05 both branch from 03 and can run in parallel. Before Stage 6
  starts, 05 is rebased onto 04.
- Stage 7 was done early on 00. Before Stage 8 starts, it is rebased onto
  06.
- Stage 1 only adds a script, so nothing builds on its branch. It
  provisions the PostgreSQL that Stages 3, 6 and 8 to 10 run against.

## Goal and scope

Bootstrap a Juju 4 controller, built from this tree, whose controller and
model databases live in an external PostgreSQL. The bootstrap is selected
by controller config, for example:

```sh
$(go env GOPATH)/bin/juju bootstrap lxd pg-poc --build-agent \
  --config db-backend=postgresql \
  --config db-url="$JUJU_PG_POC_DB_URL"
```

In scope (D1):

- LXD machine controllers, including `enable-ha`.
- Fresh bootstrap only.
- Externally provisioned PostgreSQL reached via a URL.
- dqlite remains the default and keeps working.

Out of scope for the PoC:

- Kubernetes controllers.
- Migrating an existing dqlite controller to PostgreSQL.
- `dbrepl` / `dbreplaccessor` on PostgreSQL.
- Credential rotation, the controller-charm relation and production
  hardening.
- Performance work (see ANALYSIS.md section 10).

## Decision log

All confirmed with the user. Rationale and alternatives are in ANALYSIS.md.

| ID | Decision |
|---|---|
| D1 | Scope as above: LXD machine controllers plus HA, fresh bootstrap, external PG via URL. |
| D2 | A separate Juju 3.6 controller (name passed in, never hardcoded) deploys `postgresql` 16/stable and `data-integrator` (`database-name=juju`). The PoC uses the scoped user and database this produces, so it has the same privileges a future relation would grant. |
| D3 | One PG database; one PG schema per namespace (`controller`, one per model). One `*sql.DB` per namespace with `search_path` in the connection config. Keep a namespace to (cluster, database, schema) placement indirection so models can be sharded later. |
| D4 | Backend chosen at runtime from agent.conf. Both backends compiled in; dqlite is the default. A separate PG dbaccessor worker is selected in the manifold. |
| D5 | pgx v5 `stdlib` behind an in-tree wrapping `database/sql` driver that rewrites `@sqlair_N` and `?` to `$N`. Long term: upstream sqlair dialect support. |
| D6 | Move the existing schema to `domain/schema/dqlite/` as a mechanical commit; add `domain/schema/postgresql/` generated by a reusable translator with checked-in per-patch overrides; `triggergen` gains a PG mode. Hand-written translation is acceptable for the hardest parts. |
| D7 | SERIALIZABLE isolation everywhere, retry on 40001 / 40P01. change_log writes serialised per namespace with `pg_advisory_xact_lock` in trigger functions. Alternatives documented for later. |
| D8 | TLS is part of the long-term design. The provisioning script deploys plain PostgreSQL by default (`sslmode=disable`) and has an opt-in TLS mode (`self-signed-certificates` on `client-certificates`) that saves the CA to a file and emits `sslmode=verify-full&sslrootcert=<path>`, so TLS overhead can be measured against plain. |
| N1 | Config keys `db-backend` (`dqlite` default, or `postgresql`) and `db-url` (required iff `postgresql`). They flow controller config -> instancecfg -> agent.conf like `dqlite-busy-timeout`. The URL is visible in controller config; acceptable for the PoC only. |
| N2 | Connection details are read through a "connection source" interface (agent.conf backed for now) so a later relation/secret source with credential rotation can replace it. |
| N3 | Plans live in `docs/implementation-plans/postgresql/`, excluded from the Sphinx build. |
| N4 | Branching as described under [branching](#branching): integration branch plus stacked `pg-poc/NN-<slug>` branches. |
| D9 | The PostgreSQL CA is public, so it is carried in a `db-ca-cert` controller config key (PEM) for the PoC; a relation may supply it long term. `db-backend`, `db-url` and `db-ca-cert` are fixed at bootstrap. |
| D10 | The dqlite schema keeps its declaration order. `pgschemagen` resolves forward references (which SQLite accepts and PostgreSQL rejects) with an ordering pass over the generated DDL (Stage 4, step 2). Reordering dqlite patches cannot work alone, because model patches depend on each other cyclically (ANALYSIS.md section 4). |
| N5 | Provisioning tooling lives in `scripts/postgresql-poc/`. |
| N6 | Opt-in PostgreSQL tests read the DB URL from `JUJU_POSTGRESQL_TEST_URL`. |
| N7 | The provisioning script is Python using only the standard library (it shells out to `juju ... --format=json`), runnable with plain `python3`. |
| N8 | The stages form one linear stack rather than a merged graph, with at most two stages in flight: 00 -> 02 -> 03 -> {04, 05} -> 06 -> 07 -> 08 -> 09 -> 10. 03 was rebased onto 02 without conflicts; 05 is rebased onto 04 before Stage 6, and 07 (done early on 00) onto 06 before Stage 8. Stage 1 stays a standalone branch on 00. |
| D11 | Schema selector (open question 1): `domain/schema` has `type Backend string` (`BackendDqlite`, `BackendPostgreSQL`, values matching `db-backend`) and `ControllerDDLForBackend` / `ModelDDLForBackend(Backend) (*schema.Schema, error)`; `ControllerDDL()` / `ModelDDL()` stay dqlite. `core/database/schema` has a `Dialect` carried by `Schema`, so the bookkeeping table is dialect-aware without callers knowing. Stage 8 wires the selector into bootstrap, the dbaccessor tracker and the upgrade worker. |
| D12 | `pgschemagen` reads the complete dqlite patch list (`ControllerDDL()` / `ModelDDL()`, SQL files plus Go-built triggers) and translates every trigger generically to a PL/pgSQL function and trigger. `triggergen` gets no PostgreSQL mode and the Go-built triggers are not ported by hand (replaces Stage 4 steps 3 and 4). |
| D13 | change_log ordering (D7): one `BEFORE INSERT` trigger on `change_log` takes `pg_advisory_xact_lock` keyed on the table's schema, then assigns `NEW.id` from the sequence, so ids are drawn after the lock and become visible in commit order for every writer. |
| D14 | Types: `INT` / `INTEGER` -> `bigint`; `AUTOINCREMENT` -> `bigint GENERATED BY DEFAULT AS IDENTITY`; `DATETIME` / `TIMESTAMP` -> `timestamptz`; `STRFTIME` / `DATETIME('now')` -> `now()`; `BOOLEAN` -> `boolean`; `BLOB` -> `bytea`; `user` is quoted. Identifiers over 63 bytes are shortened with a hash suffix. |
| D15 | Generated, checked-in `domain/schema/postgresql/<ns>/sql/NNNN-<slug>.sql`, one per dqlite patch, embedded by `domain/schema`. A hand tweak lives in `<ns>/overrides/` as `NNNN-<slug>-auto.sql` (raw translator output, rewritten every run) plus a hand-maintained `NNNN-<slug>.diff` (unified diff, auto -> desired) applied before the ordering pass. A hunk that no longer applies, or a diff with no patch, fails generation. A test checks the generated files are up to date. |
| D16 | Stage 5 query fixes: lease and lease-expiry times are computed in Go and bound as `YYYY-MM-DD HH:MM:SS` UTC text (the values SQLite stored before); plain "now" columns use `CURRENT_TIMESTAMP`; `GROUP_CONCAT` in `GetUnitsK8sPodInfo` is replaced by aggregating rows in Go, with no dialect hook; `PRAGMA defer_foreign_keys` is left for Stage 4 / 6 (recorded in Stage 5 findings). |
| D17 | PostgreSQL constraint tag, space and zone tables get explicit identity `rowid` columns, preserving the insertion order used by `v_application_constraint`. Keep inserts column-qualified. The dqlite definitions are unchanged. |
| D18 | `v_revision_updater_application_unit` groups by `a.uuid`, returning one actual unit count per application. This initially required a PostgreSQL override for the dqlite `GROUP BY u.uuid` bug. At the user's request, Stage 4 subsequently incorporated the shared schema fix and removed the obsolete override. The current 4.1 beta base schema changes in place; supported 4.0-to-4.1 moves use a fresh controller and model migration, not an in-place controller upgrade. |

## Environment

- `juju` on `PATH` is Juju 3.6. `juju_4` (snap) is a Juju 4 reference for
  comparing behaviour with dqlite only.
- PoC binaries are built from this tree: `make juju jujuagentd`. Always
  invoke the client by explicit path, `$(go env GOPATH)/bin/juju`, to avoid
  collisions with the snaps. `bootstrap --build-agent` runs
  `make jujuagentd` in this tree.
- The Juju 3.6 controller hosting PostgreSQL is a parameter (for example
  `PG_HOST_CONTROLLER`). Never hardcode controller names in scripts or docs.
- Juju 3.6 and Juju 4 clients may share `~/.local/share/juju`; take care when
  switching controllers.
- Read third-party source (charms, libraries) from local clones under
  `~/dev/go/src/github.com/<owner>/<repo>`, beside this repository, rather
  than through the GitHub API. Clone there if missing. Already cloned:
  `canonical/postgresql-operator` (16 charm on branch `16/edge`),
  `canonical/postgresql-single-kernel-library` (the charm's logic) and
  `canonical/data-integrator`. Go module sources are in the module cache.
- `psql` is installed on the host.
- Agent sessions see `~/.local/bin` on `PATH`, but not `~/go/bin`. The lint
  tools for `STATIC_ANALYSIS=1 make pre-check` (golangci-lint 2.14.0,
  `gci`, `govulncheck`, `go-licenses`) are in `~/.local/bin`; `shellcheck`
  is in `/usr/bin`. The `go` snap provides Go 1.27, so its `gofmt` matches
  the `go.mod` toolchain.

## Secrets

- Never commit a DB URL or password.
- Keep the URL in an environment variable (`JUJU_PG_POC_DB_URL`) or a file
  outside the repository with mode 0600.
- Scripts print credentials only to stdout or a file the user names.

## Branching

Confirmed (N4):

- `juju-4-postgresql` is the integration branch and holds this plan.
- `pg-poc/00-base-fixes` sits on top of `juju-4-postgresql` and holds only
  code fixes for pre-existing problems; its notes are under
  [Base fixes](#base-fixes). Never commit stage work to it.
- Each stage gets a branch `pg-poc/NN-<slug>`, stacked on the branch below
  it in the dependency graph (N8):
  - `pg-poc/01-provision` and `pg-poc/02-schema-move` branch from
    `pg-poc/00-base-fixes`; 03 from 02; 04 and 05 from 03.
  - The Stage 6 session first rebases 05 onto 04 (in the 05 worktree,
    `git rebase --update-refs pg-poc/04-schemagen`), then branches 06
    from 05.
  - The Stage 8 session first rebases 07 onto 06 (in the 07 worktree,
    `git rebase --update-refs pg-poc/06-test-harness`), then branches 08
    from 07.
  - 09 branches from 08, and 10 from 09.
  - Until then, `pg-poc/07-config` stays on `pg-poc/00-base-fixes`.
- Restack after every plan commit, in this order:
  1. `git rebase juju-4-postgresql pg-poc/00-base-fixes`, then
     `git switch juju-4-postgresql` (no worktree has the base branch
     checked out).
  2. In each open stage branch's worktree, bottom of the stack first,
     `git rebase --update-refs <base>`, where `<base>` is the branch below
     it. Git skips the superseded copy of the base commit because it is
     already applied. Leave a worktree with uncommitted changes to its own
     session.
- Stage 2 (schema move) must stay purely mechanical so it can be reviewed,
  and possibly proposed upstream, on its own.

## Base fixes

Branch `pg-poc/00-base-fixes`, cut from `juju-4-postgresql`, fixes
pre-existing problems unrelated to the PoC. All stages build on it.

- `go.mod` pins `go 1.27.2`, and `domain/removal/service` uses generic
  methods, which need Go 1.27. `go build ./...` passes. The "method must
  have no type parameters" errors (seen in several stage sessions) came
  from an older `gofmt` on `PATH` (snap Go 1.24), not from the build. Use
  the Go 1.27 `gofmt`.
- With Go 1.27, `gofmt -l` listed only the generated trigger files; the
  other files were Go 1.24 `gofmt` artefacts. Fixed: `generate/triggergen`
  formats its output with `go/format`, and the trigger files are
  regenerated. `go generate ./domain/schema/...` must leave no diff.
- `go vet ./generate/...` flags a json tag on an unexported field in
  `generate/schemagen/jsonschema-gen/reflect_test.go`. It is intentional
  (the test checks such fields are ignored) and suppressed for
  golangci-lint with `//nolint:govet`, so it is left as is. Plain
  `go vet ./...` is not a gate in this repository.
- `make pre-check` runs static analysis only with `STATIC_ANALYSIS=1`. It
  needs golangci-lint 2.14 exactly, and `shellcheck` on `PATH`.

## Stages

### Stage 0: plan documents

- Goal: review and agree this plan and ANALYSIS.md.
- Exit criteria: the user approves; the plan is committed on
  `juju-4-postgresql` (`docs:` commit).
- Status: in review.

### Stage 1: PostgreSQL provisioning script

- Goal: a repeatable way to obtain a `db-url` for a PostgreSQL that LXD
  controllers can reach.
- Prerequisites: a Juju 3.6 controller on LXD.
- Steps:
  1. Add `scripts/postgresql-poc/deploy-postgresql.py` (N7) with
     `-c/--controller` and `-m/--model` (both required), an optional
     `-o/--output <file>` for the URL and an optional
     `-t/--tls-ca-file <file>` for TLS mode (D8).
  2. Deploy `postgresql --channel 16/stable` and
     `data-integrator --config database-name=juju`; integrate them; wait for
     active/idle. In TLS mode also deploy `self-signed-certificates` and
     integrate it with `postgresql:client-certificates`.
  3. Run the data-integrator `get-credentials` action and build
     `postgres://<user>:<password>@<host>:5432/juju?sslmode=...`
     (`disable`, or `verify-full&sslrootcert=<ca-file>` in TLS mode).
  4. Document teardown (`juju destroy-model`).
- Verification:
  - Connect with `psql` (or a small Go probe) from the host and from a fresh
    LXD container on the same bridge.
  - Confirm the user can `CREATE SCHEMA` and `DROP SCHEMA` in `juju`.
  - Note whether `pg_hba` admits the LXD subnet; if not, record the
    workaround.
  - Repeat the connection checks in TLS mode with `verify-full`.
- Open questions: none (resolved by D8, N5 and N7).
- Status: in review. The script is committed on `pg-poc/01-provision` and
  all verification above passed.
- Restart notes:
  - Verified on 2026-10-23 against a Juju 3.6.29 LXD controller:
    postgresql 16/stable rev 1218 (PostgreSQL 16.15), data-integrator
    latest/stable, self-signed-certificates 1/stable. A fresh deployment
    settles in about 5 minutes. Re-running the script is a no-op that
    rewrites the same URL.
  - Running deployments on `$PG_HOST_CONTROLLER`: model `pg-poc` (plain) and
    model `pg-poc-tls` (TLS). URL and CA files live outside the repository
    (the agent used `~/.config/juju-pg-poc/`); re-run the script to
    regenerate them.
  - Next: user review of the script, then Stage 2, 3 or 7.
- Findings (for later stages):
  - `pg_hba` has `host juju relation-N 0.0.0.0/0 scram-sha-256` (`hostssl`
    with TLS). The host (via the `lxdbr0` gateway) and LXD containers on
    the bridge connect; no workaround is needed.
  - The login user (`relation-N`; note the hyphen when quoting) is switched
    to `charmed_juju_owner` at login by the charm's `login_hook` extension.
    New schemas and tables are owned by `charmed_juju_owner`; the database
    is owned by `charmed_databases_owner`. `RESET ROLE` and `DISCARD ALL`
    revert to `relation-N` and lose the owner privileges, so the Stage 3
    driver and Stage 8 pools must not reset the session role, or must run
    `SET ROLE charmed_juju_owner` after connecting.
  - The `juju` database already has `public` and `login_hook` schemas;
    namespace schema names must not collide with them (Stage 8).
  - The server default isolation is `read committed`; D7 relies on the
    connection config (Stage 3, step 5).
  - Enabling TLS makes `pg_hba` `hostssl` only, so plain and TLS need
    separate models. Server certificates carry the unit IP as an IP SAN, so
    `verify-full` against the IP works. The self-signed CA is valid for one
    year. `sslmode=disable` and a wrong CA are both rejected.
  - Verification used `psql` with the URL split into `PG*` environment
    variables, keeping the password out of process arguments.

### Stage 2: move schema to `domain/schema/dqlite`

- Goal: separate the dqlite DDL so a PostgreSQL schema can sit beside it.
  Mechanical, with no behaviour change.
- Steps:
  1. `git mv domain/schema/{controller,model}` (SQL, triggers, release
     DDL) under `domain/schema/dqlite/`, keeping the existing Go API
     (`schema.ControllerDDL()`, `schema.ModelDDL()`) as thin wrappers.
  2. Update `go:generate` directives, `generate/triggergen`, embed paths and
     `domain/schema/AGENTS.md`.
- Verification: `go generate ./domain/schema/...` produces no diff;
  `go test ./domain/schema/... ./core/database/...` passes; schema hashes are
  unchanged.
- Status: in review. Committed on `pg-poc/02-schema-move` (one `refactor`
  commit); verification passed.
- Restart notes:
  - Done 2026-10-23: `git mv` of `domain/schema/{controller,model}` to
    `domain/schema/dqlite/`; embed paths, `go:generate` destinations,
    trigger package imports, `generate/ddlgen`, `.github/workflows/ddl.yml`
    and `context-tests.yml`, comments and `domain/schema/AGENTS.md` updated.
    The Go package stays `domain/schema` with `ControllerDDL()` and
    `ModelDDL()` unchanged, so no callers changed.
  - `go generate ./domain/schema/...` left no diff; `go test
    ./domain/schema/... ./core/database/... ./domain/agentbinary/...
    ./domain/modelmigration/...` passes (the schema suites cover hashes).
  - The branch is based on `pg-poc/00-base-fixes`; Stage 3 stacks on it.
  - Next: user review.
- Findings (for later stages):
  - Stage 4 should add `domain/schema/postgresql/` beside `dqlite/`; the
    `schema` package (`controller.go`, `model.go`, `patch.go`) still owns
    the dqlite embed and `readPatches`, and will need a backend selector
    (open question 1).
  - Historic `.ddl` release files contain stale path comments; left as is.
  - The build errors and `gofmt` noise seen during this stage were a Go
    1.24 `gofmt` on `PATH` plus ungofmt-ed generated trigger files; both
    are resolved by Stage 00 (see [Base fixes](#base-fixes)).

### Stage 3: PG driver, placeholder rewriting, errors

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
  commit, rebased onto `pg-poc/02-schema-move`). Unit and opt-in
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
  - Next: user review, then Stage 4 or 5 (both also need Stage 2 / 3).
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
    `P0001` (plain `RAISE EXCEPTION`), so Stage 4's `triggergen` PG mode
    should use the default errcode. `ErrConstraintRowID` has no PG
    equivalent and `database.IsErrConstraintRowID` is unchanged, and
    `database.IsExtendedErrorCode` (generic SQLite error) is not extended.
  - `drivererrors.IsError` / `IsErrorTarget` now include `*pgconn.PgError`,
    so `domain` error masking hides PG errors like SQLite ones.
  - `go mod tidy` added `pgx/v5` v5.11.0 plus `pgpassfile`, `pgservicefile`
    and `puddle/v2`.
  - Statement caching: pgx stdlib defaults to cached prepared statements;
    Stage 6 should watch for "cached plan must not change result type"
    errors when DDL changes under a live pool.

### Stage 4: PG schema generation

- Goal: generate `domain/schema/postgresql/` from the dqlite patches with a
  reusable tool.
- Steps:
  1. `generate/pgschemagen`: translate patch by patch (types, identity
     columns, time defaults, `IIF`, generated columns, quoting `"user"`).
     Per-patch override files hold hand-written translations.
  2. Ordering pass in `pgschemagen` (D10), run over the whole ordered
     statement list of each namespace after translation and overrides.
     PostgreSQL resolves `REFERENCES` targets and view sources at `CREATE`
     time; SQLite does not (ANALYSIS.md section 4).
     - Strip each `FOREIGN KEY` whose target table, or the unique index it
       relies on, does not exist yet, and emit it as `ALTER TABLE ... ADD
       CONSTRAINT ... FOREIGN KEY`.
     - Move each view whose sources do not exist yet.
     - Append relocated statements to the end of the patch that creates
       their last dependency, after that patch's seed rows, so the patch
       count matches dqlite.
     - Leave tables, indexes, triggers and seed rows in place. No trigger
       is defined on a view, and PL/pgSQL bodies are not resolved at
       `CREATE` time.
     - Fail generation on dangling references and view cycles.
     This is safe because `Schema.Ensure` applies all patches in one
     transaction and PostgreSQL DDL is transactional.
  3. Trigger translation in `pgschemagen` (D12): each `CREATE TRIGGER`
     becomes a PL/pgSQL function and trigger, with `RAISE EXCEPTION` and
     the `WHEN` condition in the body. A `BEFORE INSERT` trigger on
     `change_log` takes the per-namespace `pg_advisory_xact_lock` (D13).
  4. The Go-built triggers (`domain/schema/triggers.go`,
     `modeltriggers.go`, `triggergen` output) are translated by step 3
     from the dqlite patch list; `triggergen` is unchanged.
  5. Make the bookkeeping in `core/database/schema` dialect-aware
     (`query.go`: identity column, timestamps, placeholders) (D11).
  6. Add a backend-aware schema selector (D11). Stage 8 uses it in
     bootstrap, the tracker and the upgrade worker.
- Verification:
  - Unit tests for the ordering pass: a forward FK, a forward view, an FK
    whose target unique index comes later, a two-table FK cycle (both
    become `ALTER TABLE`), and failures on a dangling reference and a view
    cycle.
  - Apply controller DDL and model DDL to empty PG schemas.
  - Table, view, index and trigger counts match dqlite (allowing documented
    differences).
  - Seed rows match row for row.
- Status: done, in review. Code on `pg-poc/04-schemagen`:
  `7d29675904` (`feat(database): support postgresql schema bookkeeping`)
  and `ca68f6d602` (`feat(schema): generate and embed postgresql ddl`).
  These replace the WIP commit; no branch was rebased or restacked.
  Follow-up commits are `57d29726a2` (`fix(schema): correct lookup types
  and revision unit counts`) and `fe5d42334e` (`refactor(schema): remove
  redundant postgresql overrides`).
- Restart notes:
  - Done: generic translator, trigger functions, forward-FK/view ordering,
    checked-in unified-diff overrides, freshness checks, embedded backend
    selectors, dialect-aware bookkeeping and documentation. There are 61
    controller and 156 model generated patches (zero-based file names;
    schema bookkeeping versions remain one-based).
  - `ControllerDDL()` / `ModelDDL()` still select dqlite. Stage 8 must use
    `ControllerDDLForBackend` / `ModelDDLForBackend` at runtime; no callers
    were changed here. `schema.NewForDialect(schema.PostgreSQL, ...)` is
    used by the PostgreSQL selectors.
  - Verified against PostgreSQL 16.15 on `lxd36-pgtest:pg-poc`: fresh
    controller/model application, idempotent reapplication, hash mismatch
    detection, rollback, foreign-key column counts, seed rows, every view's
    queryability, trigger guards, change events and blocked-writer commit
    ordering. Seed timestamps are checked against the application interval
    rather than compared literally across separately applied databases.
  - `go test -race ./core/database/schema ./domain/schema/...
    ./generate/pgschemagen ./domain/modelimport/... ./domain/export/...`
    passes with `JUJU_POSTGRESQL_TEST_URL` set.
    Bounded stress with two processes completed six runs without failures;
    the two schemas left by termination of the final active runs were
    explicitly removed. Tests otherwise drop their throwaway schemas.
  - `go generate ./domain/schema/...` and
    `go generate ./generate/export ./generate/modelimport` leave no diff.
    Changed-package `golangci-lint` reports zero issues; `make pre-check`
    passes (its repository-wide static analysis is opt-in and was not
    enabled).
  - Next: user review. No Stage 5 changes were made. The branch remains on
    its original Stage 3 base, `5135f22d3e`, by explicit instruction not to
    rebase or restack. Coordinate subsequent integration separately.
  - At the user's request, incorporated the shared dqlite cleanup from
    the other context: eleven lookup-column type corrections, regenerated
    export types, explicit legacy numeric-string import conversions and
    revision-updater grouping by application. Regression coverage checks
    integer persistence and import/export, nullable IDs, invalid input and
    zero/multiple-unit application counts.
  - Removed obsolete PostgreSQL override pairs and the redundant hunks
    in retained overrides. Regeneration leaves the final PostgreSQL SQL
    unchanged; remaining overrides still handle dialect-specific cases.
  - These corrections edit the current 4.1 beta base schema. The supported
    4.0-to-4.1 path bootstraps a new controller and migrates models through
    the versioned import transform; historic 4.0 payload types are
    unchanged. Recreate existing beta PoC databases because dqlite patch
    hashes change. The generator intentionally fails on `.PATCH.sql`
    files, so support for future shared upgrade patches needs explicit
    follow-up rather than silently omitting them.
- Findings (for later stages):
  - Object counts, including the bookkeeping table/index and excluding
    SQLite internal objects and PostgreSQL constraint indexes:

    | Namespace | Tables | Views | Explicit indexes | SQLite / PG triggers |
    |---|---:|---:|---:|---:|
    | controller | 95 | 20 | 90 | 89 / 90 |
    | model | 251 | 59 | 193 | 291 / 292 |

    The extra trigger in each PostgreSQL schema is the D13 advisory-lock
    trigger. IDs are assigned after acquiring the transaction lock; gaps
    from the earlier identity default and rolled-back transactions are
    expected. Do not assume contiguous IDs.
  - `ANALYSIS.md` section 4 describes an older source tree: all 380 triggers
    here are Go-built (not SQL-file declarations); there are no generated
    columns, REAL columns, GLOB or GROUP_CONCAT in the source DDL. Declared
    types include 258 INT, 8 INTEGER, 38 DATETIME, 24 TIMESTAMP, 42 BOOLEAN
    and 2 BLOB columns. Some trigger names exceed PostgreSQL's 63-byte
    limit; the generator adds a stable hash suffix.
  - Eleven TEXT-to-integer foreign-key mismatches in dqlite were masked by
    SQLite coercion. The shared schema now corrects `cloud_credential.auth_type_id`,
    `controller_node_password.password_hash_algorithm_id`, `autocert_cache.encoding`,
    `model_life.life_id`, `secret_permission.scope_type_id`, `charm_config.type_id`,
    `machine.password_hash_algorithm_id`, `machine_platform.os_id`,
    `application_platform.os_id`, `application_agent.password_hash_algorithm_id`
    and `unit.password_hash_algorithm_id`. PostgreSQL translates these
    directly to BIGINT; no type-correction overrides remain.
  - Other overrides make predicate-free singleton joins explicit, aggregate
    the latest-login timestamp, implement D17, compare the TEXT relation
    status to `'1'`, and disambiguate duplicate foreign-key constraint names
    on `application_remote_consumer`. `relation_status_type.id` and
    `relation_status.relation_status_type_id` both remain TEXT, matching
    dqlite; Stage 6 should check parameter binding at those callers.
  - The ordering pass preserves patch count, leaves tables/seeds/triggers in
    place and appends deferred FKs/views after the last dependency's seeds.
    Unit tests also cover FK cycles, unique-index targets, dangling
    references, view cycles, malformed/stale diffs and literal preservation.
  - Stage 5's `PRAGMA defer_foreign_keys` sites remain a Stage 6 integration
    concern. PostgreSQL foreign keys are not made deferrable by this stage.

### Stage 5: query portability fixes

- Goal: make shared queries valid on both backends.
- Steps (locations in ANALYSIS.md section 5):
  1. `IFNULL` to `COALESCE`.
  2. `INSERT OR IGNORE` to `ON CONFLICT DO NOTHING`.
  3. Replace `GROUP_CONCAT` (aggregate in Go or dialect hook).
  4. Lease and change stream time arithmetic: compute in Go and bind.
  5. Quote `"user"` in queries.
- Verification: existing dqlite tests still pass; the same tests pass on PG
  once Stage 6 lands.
- Status: in review. One `fix(domain)` commit on `pg-poc/05-queries`
  (based on `pg-poc/03-driver`). Affected domain, changestream and
  leaseexpiry suites pass on dqlite/SQLite; nothing was run on PG (needs
  Stages 4 and 6). `golangci-lint` not run.
- Restart notes:
  - Done: `IFNULL` -> `COALESCE` (5 sites); `INSERT OR IGNORE` -> `ON
    CONFLICT DO NOTHING`; `"user"` quoted in every query and test helper
    SQL found (credential, access, modelmigration, model, keymanager, ssh);
    `user` alias in `getUserWithAuth` renamed to `u`; `GetUnitsK8sPodInfo`
    aggregates ports in Go (see D16).
  - Time: lease `ClaimLease` / `ExtendLease` bind `start_at` / `expiry_at`
    (`YYYY-MM-DD HH:MM:SS` UTC, expiry rounded up to whole seconds) from a
    `clock.Clock` field on `lease/state.State` (wall clock, no constructor
    change); `leaseexpiry` binds the same format for its `expiry < ?`
    checks. Plain `DATETIME('now')` sites (removal storage status x2,
    upgrade `node_upgrade_completed_at`, change stream watermark x2) use
    `CURRENT_TIMESTAMP`, which SQLite renders identically, to avoid
    threading a clock through three state constructors.
  - Not done on purpose: `core/database/schema/query.go` (Stage 4);
    `PRAGMA defer_foreign_keys` (see findings).
- Findings (for later stages):
  - Bound lease timestamps are zone-less text. With D14 (`timestamptz`) PG
    interprets them in the session `TimeZone`, so the connection must run
    with `TimeZone=UTC` (Stage 3 driver does not set it yet; Stage 6 or 8
    should add a startup parameter) or leases expire at the wrong time.
    `CURRENT_TIMESTAMP` is fine either way.
  - `PRAGMA defer_foreign_keys = 1` in `domain/recovery/state/load.go` and
    `domain/modelimport/state/model/import.go` has the PG equivalent `SET
    CONSTRAINTS ALL DEFERRED`, which only affects `DEFERRABLE` foreign keys.
    Stage 4 would have to emit FKs as `DEFERRABLE INITIALLY IMMEDIATE`
    (including D10's `ALTER TABLE ... ADD CONSTRAINT` ones). Not needed for
    bootstrap. Other SQLite-only code left for Stage 6/8: `pragma.*`
    (`foreign_keys`, `delete_database`), and `""` used as an empty string
    literal in `domain/model/state/testing/model.go` (`VALUES (?, ?, ?, "",
    true)`), which PG reads as an identifier.
  - Sites that look portable but are only checked on SQLite: SQLite lets
    `GROUP BY` leave columns bare and compares int to text freely; Stage 6
    will surface more.
  - Next: user review, then Stage 6 (needs Stages 4 and 5).

### Stage 6: PG test harness for domain state tests

- Goal: run selected domain state suites against PostgreSQL to flush out
  type-affinity and dialect bugs early.
- Steps: a PG-backed equivalent of `internal/database/testing` and
  `domain/schema/testing` that creates a throwaway schema per test, enabled
  by the test env var. Start with lease, changestream, controllernode,
  model, application and machine.
- Verification: the chosen suites pass on both backends; known failures are
  listed here.
- Restart notes: none yet.

### Stage 7: config plumbing

- Goal: `db-backend` and `db-url` reach every controller's agent.conf.
- Steps:
  1. Controller config keys and validation (`controller/config.go`).
  2. instancecfg / agent.conf fields (`agent/agent.go`,
     `agent/format-2.0.go`).
  3. Bootstrap and `agentconfigupdater` propagation.
  4. Connection-source interface (N2).
- Verification: bootstrap a dqlite controller with the keys unset (no
  change); unit tests round-trip the keys through agent.conf.
- Open questions: how the PostgreSQL CA reaches every controller for
  `sslmode=verify-full` (D8). `sslrootcert` is a file path, so the CA must
  be carried in config (for example a `db-ca-cert` key) and written out on
  each controller, or the connection source must build the TLS config
  directly. **Resolved:** a public `db-ca-cert` config key (PEM); a later
  relation can replace it.
- Status: in review. Committed on `pg-poc/07-config` (7ebc21b74a).
- Restart notes:
  - Done: controller keys `db-backend`, `db-url`, `db-ca-cert` (immutable,
    in `ControllerOnlyConfigAttributes` only, validated: `db-url` required
    iff postgresql); `agent.Config` `DBBackend/DBURL/DBCACert` and
    `SetDBConfig`, stored in agent.conf (`dbbackend`, `dburl`,
    `dbcacert`); set from instancecfg (new/HA machines) and in
    `cmd/jujuagentd/agent/bootstrap.go`; `internal/database/connsource`
    (`Source`, `Details`, `NewAgentConfigSource`). Agent mocks regenerated.
  - Not done, by design: `agentconfigupdater` does not propagate the keys
    because they are fixed at bootstrap (a change must not restart agents);
    CAAS `podcfg` / `caasapplication` do not set them (out of scope, D1).
    The sslrootcert file is not written; Stage 8 should write
    `Details.CACert` to a file under the data dir when building the pgx
    config (or set the TLS config directly).
  - Not run: a live dqlite bootstrap; unit tests pass (agent, controller,
    instancecfg, connsource, jujuagentd/agent). `gci` was not installed, so
    imports were ordered by hand.
- Findings: Stage 8 should read connection details only through
  `connsource.Source`. `db-url` contains the password and is stored in
  controller config and agent.conf (N1).

### Stage 8: PG dbaccessor worker and bootstrap

- Goal: controller agents open namespaces on PostgreSQL.
- Steps:
  1. A PG `DBApp`/worker selected in the dbaccessor manifold when
     `db-backend=postgresql`. It needs no node lifecycle or handover.
  2. `ensureNamespace`: `CREATE SCHEMA IF NOT EXISTS`, then a pool with
     `search_path`; the tracker applies model DDL lazily (replace the
     `sqlite_master` check and the `PRAGMA`s). Schema names must not
     collide with the existing `public` and `login_hook` schemas.
  3. `DeleteDB`: `DROP SCHEMA ... CASCADE`.
  4. `BootstrapPostgres` beside `BootstrapDqlite`; `controller_node` rows
     carry no dqlite IDs.
  5. Leave `ClusterDescriber` and the HA hooks as no-ops for PG.
- Verification: a bootstrap-only integration test against PG; the
  controller namespace and the first model namespace are created.
- Restart notes: none yet.

### Stage 9: LXD bootstrap and smoke test

- Goal: end-to-end proof.
- Steps:
  1. `make juju jujuagentd`.
  2. Bootstrap as shown under [Goal and scope](#goal-and-scope).
  3. Smoke test: `add-model`, deploy a simple charm, `status`,
     `remove-application`, `destroy-model` (schema dropped).
  4. Run the same steps with `juju_4` on dqlite and compare.
- Verification: the smoke test passes; findings are recorded here.
- Restart notes: none yet.

### Stage 10: enable-ha

- Goal: three controllers sharing the PostgreSQL backend.
- Steps: `enable-ha -n 3`. Confirm the new nodes get the backend config,
  leases behave, change streams deliver on all nodes and witness rows prune
  correctly.
- Verification: kill the API leader controller; the model stays operable.
- Restart notes: none yet.

### Stage 11: future work (not planned)

- Inception: a dqlite controller deploys PostgreSQL, then migrates via
  model export/import or a schema-aware copy.
- A controller charm relation to `postgresql` via `postgresql_client`, with
  credentials from a secret and rotation.
- Concurrency and change-stream alternatives (ANALYSIS.md section 10).
- Placement indirection for sharding models across databases or clusters.
- Dual-backend CI and a check that generated PG schema is up to date.
- Upstream sqlair dialect support.

## Open questions

1. ~Exact Go API for selecting the schema by backend (Stage 4).~
   Resolved: D11.
2. ~How the PostgreSQL CA reaches every controller for TLS (Stage 7, D8).~
   Resolved: `db-ca-cert` config key.
3. ~Branch base for a stage with several dependencies, such as Stage 4
   (Stages 2 and 3): stack on one and rebase onto the other, or merge.~
   Resolved: N8, stack linearly.

## Assumptions to verify

- `enable-ha` copies `db-backend` / `db-url` to new controllers via
  controller config -> instancecfg with no extra work (Stage 10).
- Domain code tolerates SERIALIZABLE retries; no transaction body has
  non-idempotent side effects outside the DB (Stage 6).
- SQLite type affinity hides a bounded number of type mismatches that the
  Stage 6 harness will surface.
