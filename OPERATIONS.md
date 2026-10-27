# Build and validation reference

Read the relevant section when building, testing, regenerating schemas or
operating the PoC. PLAN.md keeps the always-needed safety and stack rules.
Follow the code repository's AGENTS.md and nested guidance as well.

## Exceptional stack operations

Routine new-stage/fix/restack/push commands stay in PLAN.md. For an
approved amendment, create `git commit --fixup=<sha>` from the top, then
run `GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash --update-refs <base>`.
Use the current base from PLAN.md, not a historical hash.

For a new upstream base, only when the user asks:
`git rebase --update-refs --onto <new-base> <old-base> <top>`, then update
PLAN.md. Inspect the graph and verify adjacent branch ancestry after any
restack. Stop on conflicts or unexpected duplicated/skipped commits;
[earlier recovery history](history/compatibility.md#historical-runs-and-restacking)
is diagnostic context, not a command to repeat blindly.

## Local tools

- `juju` on PATH is Juju 3.6. Use it only for the PostgreSQL hosting
  controller. Use the explicit built client path for PoC controllers.
- `juju_4` is a reference snap, not proof that the modified tree still
  works with dqlite. Benchmark baselines must identify an exact revision.
- Build with `make juju jujuagentd`; `bootstrap --build-agent` runs
  `make jujuagentd` in the code tree. `make install` includes schema
  regeneration; `make go-build` does not.
- Agent sessions see `~/.local/bin`, not `~/go/bin`, on PATH. The lint
  tools (`golangci-lint`, `gci`, `govulncheck`, `go-licenses`) were installed
  in `~/.local/bin`; `shellcheck` is in `/usr/bin`. Use the Go snap's
  `gofmt`, not an older system copy. Check versions when recording results.
- Run the repository-prescribed gci command on touched, non-generated Go
  files. Run `make pre-check`; report whether `STATIC_ANALYSIS=1` was
  enabled rather than treating the default target as full static analysis.
- Read third-party source from local clones under
  `~/dev/go/src/github.com/<owner>/<repo>` rather than the GitHub API.
  Existing clones: `canonical/postgresql-operator` (16 charm on `16/edge`),
  `canonical/postgresql-single-kernel-library`, `canonical/data-integrator`.
  Go module sources are in the module cache.

## Bounded runs and check-ins

Nothing an agent starts may run for more than about two minutes without a
progress check (N13). The go test timeout ends a hang; it is not how a
hang should be noticed.

- Tests follow the code repository's AGENTS.md: they block on an event
  that must happen, with no explicit timeout branch. A test's only bound
  is the binary's `-timeout`, which panics with every goroutine's stack.
  Use `-timeout 120s` per run, inside an outer `timeout` about 30 s
  longer.
- Split work so each run fits that budget: one package per run, and for a
  slow PostgreSQL package one suite or a group of its tests per run, for
  example `-run '^TestStateSuite$/^Test(Get|Set)'`. `tc` runs each suite
  method as a subtest. At about 2.5 s per `ModelSuite` test on
  PostgreSQL, one run holds about 40 tests. `go test -list .` lists only
  the suites; count methods in the source.
- A run that cannot be split, such as stress or a benchmark, gets an
  agreed budget. Run it in the background and check it at least every two
  minutes, reporting tests finished, the current test and failures so
  far.
- Always pass `-v` and `tee` the output to a uniquely named log, for
  example `/tmp/<pkg>-<backend>-$$.log`. Without `-v`, `go test` holds a
  package's output until the package finishes, so the log shows no
  progress.
- At each check, compare the log's latest `=== RUN` and `--- PASS/FAIL`
  lines with the previous check. A test running far beyond its usual few
  seconds is a suspected stall. Collect evidence before stopping it:
  - `pg_stat_activity` for this database: sessions waiting on a lock
    (`wait_event_type = 'Lock'`), long statements, or sessions idle in a
    transaction.
  - A goroutine dump: `kill -QUIT <pid>` on the test binary itself (the
    `<pkg>.test` child of `go test`, or a binary built with `go test -c`)
    prints every goroutine's stack and exits.

  Then record the run as inconclusive.
- PostgreSQL test sessions fail a lock wait after 30 s, a statement after
  60 s and an idle transaction after 60 s (N14), so a database-side block
  surfaces as a test error.
- Known blocking pattern: `workertest.CleanKill` / `DirtyKill` deferred in
  a test waits on `c.Context()`, which `testing` cancels only after the
  test function returns. If the worker goroutine has already ended, for
  example because a gomock expectation failed inside it, the kill blocks
  until `-timeout`. Look in the log for the gomock failure
  ("Unexpected call", "missing call(s)") above the timeout panic.

## PostgreSQL test runs

`JUJU_POSTGRESQL_TEST_URL` opts the harness into PostgreSQL (N6). Without
it, tests use the existing SQLite/dqlite path. With it, every suite built
on `DqliteSuite` is redirected, not just previously verified suites.

- Run one PostgreSQL package at a time. Do not overlap suites, stress
  workers or benchmarking on the shared server.
- Prove backend selection after harness/environment changes: run the
  selected suite with
  `postgres://nobody@127.0.0.1:1/none?connect_timeout=2` and require a
  connection failure. A previous Stage 6 run silently used SQLite.
- Use `-count=1`. Use `-race` for mutex/goroutine code and bounded stress
  for goroutine, tomb or catacomb code, as required by AGENTS.md.
- Bound every run as in [Bounded runs](#bounded-runs-and-check-ins).
  The application and removal-model packages have taken 16–17 minutes on
  PostgreSQL; run them in suite- or test-group-sized chunks, not as one
  run. A timeout is not evidence of a functional regression. The cost is
  per-test setup, not query speed (see below).
- Do not print the database URL. Obtain it from the private file or the
  environment. The snap Go command has been observed to produce no output
  with direct file redirection: use a pipe, preserving the test exit code.

Example for one small package; substitute the task's package and budgets:

```bash
set -o pipefail
export JUJU_POSTGRESQL_TEST_URL="$(cat ~/.config/juju-pg-poc/db-url)"
timeout 150s go test -count=1 -race -timeout 120s -v ./internal/database/postgresql/ \
  2>&1 | tee "$LOG_FILE"
```

### Why PostgreSQL state tests are slow

Measured 2026-10-26 on the shared test server (PostgreSQL 16.15 in LXD
container `juju-089db6-0`, ZFS pool backed by a loop file on host ext4).
The removal-model package took 67 s on SQLite and 713–983 s on
PostgreSQL. The median test time went from 0.17 s to 2.1–2.8 s.

- Each `ModelSuite` test applies the full model DDL (156 patches) to a
  fresh schema in about 2.4 s, then `DROP SCHEMA ... CASCADE` takes about
  0.4–0.5 s. Each test writes about 12 MB of WAL. A write commit takes about
  2 ms and a read-only transaction about 0.5 ms, so the test bodies
  themselves are cheap.
- The DDL time is index builds. The model DDL creates 250 tables and 688
  btree indexes (438 primary key, unique and explicit indexes, plus 250
  TOAST indexes). PostgreSQL 16 `nbtsort.c` fsyncs every WAL-logged index
  build (`smgrimmedsync`), even for an empty table. On this storage that
  costs about 3.5 ms per index, or about 2.4 s per model.
- Synthetic evidence (500 objects in one statement):

  | Objects | Time |
  |---|---|
  | Views | 0.24 s |
  | Tables, no index | 0.24 s |
  | Tables with an integer primary key | 2.09 s |
  | Tables with a text primary key | 3.89 s |
  | Temporary tables with a text primary key (no WAL or fsync) | 0.68 s |

  `synchronous_commit=off` does not change the single-transaction time,
  because these fsyncs are not commit fsyncs.
- Run-to-run drift follows fsync latency on the loop-backed pool and host
  I/O load. The catalogs have no dead tuples, and catalog-only DDL is fast,
  so catalog bloat is not the cause.
- ZFS is not the main cause. A 500-file create+write+fsync loop costs
  0.87 ms per file in the container, 0.66 ms on host ext4. In a scratch
  cluster inside the container, 500 text-PK tables took 1.15 s on ZFS,
  0.33 s with `fsync=off` and 0.26 s on tmpfs.
- Server profiles (`deploy-postgresql.py --profile`, stage 01):
  `production` (charm defaults), `testing` (default; trimmed logs and WAL,
  fsync on, no real speed-up) and `disposable` (`fsync=off`,
  `full_page_writes=off`, set through Patroni because the charm exposes
  neither; a crash can corrupt the cluster). Applied to the shared server
  on 2026-10-26: 500 text-PK tables 3.9 s -> 0.6 s. The charm rejects
  `synchronous_commit=off` and pins it in the local Patroni file, so it can
  only be set per connection.
- Remaining options:
  - Use ZFS `sync=disabled` (needs root on the host).
  - Use PostgreSQL 17 or later, whose bulk-write API passes the index sync
    to the checkpointer.
  - Use storage that is not loop-backed.
  - Change the harness, for example to clone a template database or reuse a
    schema per suite. This changes test isolation and needs a decision.

`LOG_FILE` must name a file outside both repositories. Capture the pipeline
exit status immediately in automation. For the default-backend run, unset
`JUJU_POSTGRESQL_TEST_URL` explicitly. Record build tags: an untagged
SQLite test run does not prove the actual dqlite runtime works.

For stress, compile the target package with `-race` and run the selected
suite through a bounded `stress` invocation. Use one process for PG (for
example `stress -p 1`); terminate only the specific run and account for
leftover test schemas. Do not infer a stall solely from an interval with
no log output. Inspect progress and resources before stopping a run.

## Required evidence

For each required run, record: source revision, package/suite selector,
backend and build tags, command/budget, exit status, failures and skips,
and a redacted log location. Keep three states distinct: passed at the
recorded revision, fixed but not rerun, and not run/failed.

A run interrupted by disk exhaustion or timeout is inconclusive. After a
restack, earlier passes are historical evidence, not passes of the new
revision. A skip must have an approved backend-specific reason (N10), not
merely an outstanding portability failure. Retain typed error assertions
where exposed; where state-layer masking hides driver errors, assert the
intended domain error or specific message, not just any failure.

Current Stage 6 results and remaining tasks live in
[06-compatibility.md](stages/06-compatibility.md). Session-local helper
scripts and historical logs are optional evidence, not prerequisites to
reproduce the commands.

## Schema changes

After changing shared DDL or overrides:

```sh
go generate ./domain/schema/...
```

This regenerates triggers and PostgreSQL DDL. For only PostgreSQL output,
use `go generate ./domain/schema/postgresql.go`. Freshness tests must pass.
Changes to shared column types also require the applicable export/import
regeneration and legacy-format regression tests (D30), even though running
model import/export on PostgreSQL is deferred.

Never silently recreate a live database after a schema mismatch. Current
4.1 beta schema edits require explicit recreation of disposable PoC
instances; inspect the target and get approval for destructive operations.

## Server and credentials

[Stage 1](stages/01-provision.md) owns provisioning, TLS recreation and
disk-use details. It records a prior disk-full incident and the mitigations.
The existing server has a limited connection budget; test-pool limits do
not prove enough capacity for arbitrary model counts or controller HA.

Use explicit controller/model targets because the Juju clients share local
state. Test schemas must not overlap live controller/model schemas (N9,
D20). Inspect owned schemas before cleanup; never drop all `juju_*` schemas
on an unverified target. Do not expose credentials in logs, benchmark
artifacts or process arguments.
