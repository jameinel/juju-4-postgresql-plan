# PostgreSQL PoC: handoff at the worktree collapse (2026-10-24)

The authoritative plan is
`docs/implementation-plans/postgresql/PLAN.md` on `juju-4-postgresql`. It
was updated in this session; this note summarises it and proposes the
workflow after the collapse.

## 1. What is pushed

All branches are on `origin` (`jameinel/juju`) and match local. 00 to 07
were force-pushed with leases; 08 is new.

| Branch | Tip | Contents |
|---|---|---|
| `juju-4-postgresql` | `85417735dc` | Plan, analysis, upstream fix register, D1 to D23 |
| `pg-poc/00-base-fixes` | `9ce5bebab8` | triggergen gofmt fix |
| `pg-poc/01-provision` | `d88b694235` | PG provisioning script (side branch off 00) |
| `pg-poc/02-schema-move` | `38807c7a51` | dqlite DDL moved to `domain/schema/dqlite` |
| `pg-poc/03-driver` | `55d4b64a17` | pgx driver, placeholder rewriting, errors |
| `pg-poc/04-schemagen` | `5e408c81ab` | `pgschemagen`, generated PG DDL, U1/U2 |
| `pg-poc/05-queries` | `5388681306` | Shared query portability fixes |
| `pg-poc/06-test-harness` | `a972f7c789` | Harness plus seven Stage 6 fixes (below) |
| `pg-poc/07-config` | `83fd81b16d` | `db-backend`, `db-url`, `db-ca-cert` plumbing |
| `pg-poc/08-dbaccessor` | `834cb80187` | Stage 8 bootstrap layer (two commits) |

Stack shape: integration, then 00, 02, 03, 04, 05, 06, 07 and 08, all in a
line. 01 hangs off 00. Stage 7's stray docs commit was moved to the
integration branch and dropped from 07.

## 2. Where each stage stands

### Stage 6 (reopened): PG test harness

The first pass "passed" only because `OsEnvSuite` scrubbed
`JUJU_POSTGRESQL_TEST_URL`, so every suite ran on SQLite. Fixes on 06:

| Commit | Kind |
|---|---|
| `fix(testing): keep postgresql test url through env isolation` | harness |
| `fix(database): bind controller node id as text` | latent (U3) |
| `fix(lease): make lease pinning and tests valid on postgresql` | portability, plus latent test (U4) |
| `test(schema): add backend-aware failing trigger helper` | portability |
| `test(controllernode): make address projection tests backend neutral` | portability |
| `test(changestream): insert controller node ids as text` | latent test (U5) |
| `fix(schema): keep explicit change_log ids on postgresql` | portability (pgschemagen, D13 amended) |

Verified on PG and on SQLite with `-race`: `domain/lease/state`,
`domain/controllernode/state` and `domain/changestream/state`.

Known failing or untriaged on PG:
- `internal/database` `TestMigrationSuite`. It uses `schema.New`, so the
  bookkeeping DDL is SQLite (`AUTOINCREMENT`). The harness needs a dialect
  accessor so the test can use `schema.NewForDialect`.
- Any other suite built on `DqliteSuite` also switches to PG when the
  variable is set. Only the packages above are verified.
- Not started: `domain/model/state` (controller and model),
  `domain/application/state`, `domain/machine/state`,
  `domain/sequence/state`.
- `RAISE(ABORT)` sites still to convert to `FailingTriggerSQL`:
  - `domain/removal/state/model/relation_test.go`
  - `domain/removal/state/model/relationwithremoteofferer_test.go`
  - `domain/application/state/application_refresh_test.go`
  - `domain/machine/state/reprovision_test.go`

### Stage 8 (in progress): dbaccessor and bootstrap

The design is confirmed and recorded as D19 to D23.

Done:
- `WithRootCAs`, plus `Placement` with `juju_` schemas and schema helpers.
- `DqliteInitialiser` and `PostgresInitialiser`.
- `agentbootstrap.DatabaseInitialiser`, selected in the jujud bootstrap
  command.

Tests:
- Unit tests pass on SQLite with `-race`.
- Placement and database bootstrap tests pass on PG.
- `go build ./...` is clean at the tip.

Failing: the opt-in `TestPostgresBootstrapSuite`. The seed `ON CONFLICT
DO UPDATE` queries use unqualified columns, which PG rejects as ambiguous.
The fix is portability; the sites are listed in the PLAN.md Stage 8
restart notes (user, modeldefaults, modelconfig, remoterelation, status,
sequence).

Not started:
- The tracker `WithBackend` option.
- `pgWorker` and manifold wiring.
- `upgradedatabase` DDL selection by backend.
- `scripts/postgresql-poc/drop-juju-schemas.py`.
- Stages 9 and 10.

### Upstream fix register (dqlite defects)

| ID | Kind | Item | State |
|---|---|---|---|
| U1 | bug | `v_revision_updater_application_unit` GROUP BY | fixed, Stage 4 |
| U2 | latent | Lookup FK columns typed TEXT | fixed, Stage 4 |
| U3 | latent | `InsertControllerNodeID` uint64 bind | fixed, Stage 6 |
| U4 | latent test | Lease migration seeds ignored errors | fixed, Stage 6 |
| U5 | latent test | Integer `dqlite_node_id` in tests | changestream fixed; `domain/upgrade/state` open |
| U6 | latent test | `""` literal in `domain/model/state/testing` | open |

## 3. Moving forward after the collapse (proposal, needs confirming)

The per-worktree restack existed only because Git will not rebase a branch
that another worktree has checked out. With one tree, `--update-refs` does
the whole restack in one command.

Recommended workflow:

1. Remove the stage worktrees. All are clean and pushed. `/tmp/pg06-probe`
   is a stale probe at a pre-rebase commit that no branch contains.

   ```sh
   cd ~/dev/go/src/github.com/juju/juju-4-postgresql
   for n in 02 03 04 05 06 07 08; do git worktree remove ../juju-pg-$n; done
   git worktree remove --force /tmp/pg06-probe
   ```

2. Work on the top branch, `pg-poc/08-dbaccessor`, later 09 and 10. The
   stage branches stay as review refs that `--update-refs` keeps up to
   date.
3. When a fix belongs to an earlier stage (for example more Stage 6
   suites), commit it on top, then run
   `git rebase -i --update-refs juju-4-postgresql`. Move the commit above
   the matching `update-ref refs/heads/pg-poc/06-test-harness` line. Use
   `git commit --fixup` plus `--autosquash` only to amend an existing stage
   commit.
4. Plan changes: commit on `juju-4-postgresql`, then
   `git switch pg-poc/08-dbaccessor && git rebase --update-refs juju-4-postgresql`.
   Restack 01 separately with `git rebase --onto`, or see question 1.
5. Push with `git push --atomic --force-with-lease origin <all stage branches>`.
6. Keep the classification rule (protocol step 6): bug and latent fixes go
   in their own commits and in the register, so they can be proposed
   upstream apart from the PoC.

Questions to confirm, then record in the PLAN.md Branching section:

1. Move `pg-poc/01-provision` into the line between 00 and 02 so that one
   `--update-refs` covers every branch? (Recommended: yes.)
2. Keep the plan at the bottom on `juju-4-postgresql`, or carry plan edits
   in the working branch and fold them back at review? (Recommended: keep
   it at the bottom; the restack is now one command.)
3. Order of work: finish Stage 6 suites (model, application, machine) or
   unblock the Stage 8 PG bootstrap first (seed query qualification)?
   (Recommended: the seed queries first. They are small, unblock Stage 8
   end to end, and the sequence fix overlaps the Stage 6 sequence suite.)

## 4. Working notes

- PG tests: `JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url)`.
  To prove PG is used, point the variable at
  `postgres://x:y@127.0.0.1:1/none?connect_timeout=2` and expect
  "connection refused".
- Run one package at a time with
  `timeout 200 go test -count=1 -timeout 3m -v ./pkg/`. A `SetUpTest`
  failure repeats for every test.
- The snap `go` writes nothing when stdout is redirected to a file. Pipe
  instead: `go test ... 2>&1 | cat > log`.
- PG aborts the transaction on any statement error, so resolve conflicts in
  SQL (`ON CONFLICT`) rather than catching the error.
- pgx will not bind Go integers to TEXT parameters.
- Domain errors mask driver errors, so match message text that both
  backends share.
- After regenerating with `go generate ./domain/schema/postgresql.go`, the
  `pgschemagen` tests check that the generated files are not stale.
- gci: `~/.local/bin/gci write --section standard --section default --section "Prefix(github.com/juju/juju)" <files>`.
