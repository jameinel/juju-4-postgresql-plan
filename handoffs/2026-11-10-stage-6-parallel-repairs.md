# PostgreSQL PoC: handoff, Stage 6 parallel repairs (2026-11-10)

## Objective

Clear the test-side failures from the 2026-11-10
[PostgreSQL re-sweep](../stages/06-compatibility.md#postgresql-re-sweep)
with up to four worker sessions running in parallel. Then rerun the
sweep and reassess bootstrap run 4. The re-sweep at `09=a3818782aa`
recorded 333 failing leaf tests in 35 packages. This round covers 307
of them in 31 packages; the four deferred packages are listed under
[Out of scope](#out-of-scope).

This file is for two readers: the **integrator**, a single session that
owns the `pg-poc/*` stack, the plan repository and dispatch; and every
**worker**, who reads [Worker protocol](#worker-protocol) and then its
own handoff:

| Worker | Handoff | Packages | Leaves | Batch |
|---|---|---|---:|---|
| W1 | [stream fixtures](2026-11-10-stage-6-w1-stream-fixtures.md) | changestream stream, ssh model, changestreampruner, dbrepl, dbreplaccessor | 94 | 1 |
| W2 | [removal model](2026-11-10-stage-6-w2-removal.md) | `domain/removal/state/model` | 47 | 1 |
| W3 | [application state](2026-11-10-stage-6-w3-application.md) | `domain/application/state` | 14 | 1 |
| W4 | [timestamps and `%q`](2026-11-10-stage-6-w4-timestamps.md) | status, macaroon, secretbackend, secret, operation, modelmigration | 82 | 1 |
| W5 | [bind types](2026-11-10-stage-6-w5-binds.md) | blockdevice, provisioner, storage, resource, controllerupgrader, upgrade, upgradesteps, ssh controller, crossmodelrelation | 54 | 2 |
| W6 | [errors, triggers and order](2026-11-10-stage-6-w6-errors-triggers.md) | network, machine, access, model, relation, unitstate, `domain` | 16 | 2 |

No two workers own the same package or the same test file. Stages 07–09
change none of the failing test files. Of the files in scope, only
`domain/schema/testing/schema.go` differs between 06 and 09, and only in
`SchemaApplier.Apply`. Workers therefore branch from the 09 tip, where
the sweep ran and where PostgreSQL runs of several packages at once are
safe (N17). Commits are then cherry-picked onto 06 (N18).

### Execution budget

Use normal tool parallelism defaults, adjusting for current host load.
PostgreSQL runs also share the test-server budget (N19).
Keep the bounded-run and progress checks; timeouts remain inconclusive.

### Validation update: changed layer first, full suite once

Per the user's 2026-11-10 direction, this supersedes the older full-package
requirements in the individual worker handoffs:

- During repair, run the tests affected by that layer on PostgreSQL and
  real dqlite. Include every consumer of a changed package-local fixture
  or helper, not just the originally reported failing test.
- After integration/restacking, check the affected tests at the new top.
  Do not repeat unrelated package suites, earlier layers' passing tests,
  or a whole-stack build for a test-only change. Compile the affected
  packages; broaden checks when production, shared helpers, schema or
  harness changes affect more consumers, or a conflict changes semantics.
- Keep required focused repetitions for timestamp/order repairs and
  bounded real-dqlite watcher stress. These apply to affected selectors,
  not entire packages by default.
- After all repairs are integrated and the stack is restacked, run the
  full agreed selection once per backend at that final tip. Retain the
  deferred packages, approved skips and known failures explicitly.
  If a final run exposes a defect, rerun its affected checks after the
  fix; invalidate and repeat broader results only where the fix affects
  them. Do not claim final-tip coverage from historical runs.

Existing complete runs remain useful evidence; do not repeat them merely
to satisfy the superseded per-worker procedure.

### Separate follow-up: database fixture time helper

- [x] Add `internal/testing.DatabaseNow() time.Time`, returning
  `time.Now().UTC().Truncate(time.Microsecond)`, and migrate ordinary
  current-time database fixtures introduced by these repairs.
  Document the UTC and exactly round-trippable precision contract.
  Preserve explicit non-UTC, fixed-time and precision-boundary fixtures;
  never truncate actual results to conceal precision loss.
  Keep `NonZeroTime()` unchanged. Completed on Stage 6 as `306a0ae3cf`;
  see [validation](../stages/06-compatibility.md#database-fixture-time-helper-2026-11-10).

## Starting state

- Code repository `jameinel/juju`: the main PoC tree
  `~/dev/go/src/github.com/juju/juju-4-postgresql` on
  `pg-poc/09-bootstrap`. Tips at the time of writing: `06=ebba92514b`,
  `07=fae300116a`, `08=09=a3818782aa`. Recheck them before acting.
- Plan repository: `main`. Recheck it, and preserve concurrent changes.
- Sweep evidence: session
  `~/.copilot/session-state/5a9652bd-930f-45c8-a317-a4c57e24b91c/files/`
  `pg-sweep-a3818782aa.jsonl` (structured `go test -json` output),
  `-packages.txt` and `-resolved.txt`. To list one package's failing
  leaves:

  ```bash
  J=~/.copilot/session-state/5a9652bd-930f-45c8-a317-a4c57e24b91c/files/pg-sweep-a3818782aa.jsonl
  jq -r --arg p github.com/juju/juju/domain/access/state \
    'select(.Action=="fail" and .Package==$p and .Test!=null) | .Test' "$J" |
    awk -F/ 'NF>=2' | sort
  ```

  Select `.Action=="output"` with the same package and test to read the
  failure text.

## Integrator procedure

### Step 0: trigger drop helpers

Do this before dispatching any worker. W2, W3 and W6 depend on it.

PostgreSQL's `DROP TRIGGER` needs `ON <table>`, which SQLite rejects.
On PostgreSQL `FailingTriggerSQL` also creates a trigger function, which
`DROP TRIGGER` leaves behind, so recreating the trigger fails with 42723.

1. On `pg-poc/06-test-harness`, add to `domain/schema/testing/trigger.go`:
   - `DropFailingTriggerSQL(t FailingTrigger) string`. SQLite:
     `DROP TRIGGER IF EXISTS <name>`. PostgreSQL:
     `DROP TRIGGER IF EXISTS <name> ON <table>;` followed by
     `DROP FUNCTION IF EXISTS <name>();`.
   - `DropTriggerSQL(name, table string) string`, for schema triggers
     such as `trg_model_immutable_update` on `model`. On PostgreSQL drop
     only the trigger; the function belongs to the schema.
2. Add tests in that package, for example `trigger_test.go` on a model
   schema suite, on both backends:
   - a `FailingTrigger` with and without `When` aborts the statement with
     its message;
   - after `DropFailingTriggerSQL` the statement succeeds, and the same
     trigger can be created again;
   - `DropTriggerSQL` removes a schema trigger.

   Execute the DDL through `database/sql`, not sqlair (see
   [Worker protocol](#worker-protocol), "Triggers"). The package had no
   tests in the sweep; add it to the selections below.
3. Classify the commit as `portability` with `Upstream-Home: none`
   (PoC-only backend switch). Restack under PLAN.md "Stack workflow":
   `git switch pg-poc/09-bootstrap && git rebase --update-refs
   pg-poc/06-test-harness`, then `go build ./...`.
4. Run `domain/schema/testing` and
   `domain/controllernode/state` (the existing `FailingTriggerSQL` user)
   on PostgreSQL and real dqlite. Record the revision and evidence in
   Stage 6.

### Dispatch

1. Create each worker's branch and worktree from the current 09 tip, so
   that no worker branches from a ref in the middle of a restack:

   ```bash
   N=w2 SLUG=removal
   cd ~/dev/go/src/github.com/juju/juju-4-postgresql
   git worktree add -b "pg-work/$N-$SLUG" "../juju-4-pg-$N" pg-poc/09-bootstrap
   ln -s ../juju/_deps "../juju-4-pg-$N/_deps"
   mkdir -p ~/pg-poc-work/$N
   git -C "../juju-4-pg-$N" rev-parse HEAD
   ```

   Worker branches are `pg-work/*`, outside the `pg-poc/*` atomic push.
   `_deps` is already ignored.
2. Dispatch batch 1 (W1–W4) after Step 0 is restacked. Dispatch W5 and W6
   as batch-1 workers finish, with at most four workers running. The
   prompt names the worker handoff, this file, the worktree path, the
   base commit and the `-p` budget.
3. While workers run, do not run PostgreSQL tests above the remaining
   budget, and do not bootstrap, benchmark, stress or sweep (N19).

### Integrating a worker

1. Read the worker's report. Check its commits against its scope:

   ```bash
   git --no-pager log --reverse --format='%h %s%n%(trailers)' \
     <base>..pg-work/<N>-<slug>
   git --no-pager diff --stat <base>..pg-work/<N>-<slug>
   ```

   Every touched file must be a test file (or a package-local test
   helper) in the worker's packages. Return anything else to the worker
   or the user.
2. In the main tree, with a clean status:
   `git switch pg-poc/06-test-harness`, then
   `git cherry-pick <commits>` in order. Do not use `-x`; the worker
   branches are deleted afterwards. Then
   `git switch pg-poc/09-bootstrap && git rebase --update-refs
   pg-poc/06-test-harness`.
3. Compile the affected packages and run the changed layer's affected
   tests at the new 09 tip on PostgreSQL and real dqlite, within the
   budget. Follow the validation update above rather than repeating
   full package suites after each integration. A conflict, or a failure
   that the worker's evidence did not show, goes back to the worker.
4. Fold the report into the plan:
   - [06-compatibility.md](../stages/06-compatibility.md): task rows,
     revisions and evidence;
   - [DIALECT.md](../DIALECT.md): new differences or new sites;
   - [UPSTREAM-FIXES.md](../UPSTREAM-FIXES.md): register each `bug` or
     `latent` change, and any test change tagged with an upstream home
     other than `none`, under its home. "Unassigned test findings"
     takes `new` items. Publishing any upstream PR still needs approval
     per PR.
   - Production findings: summarise them for the user. Route only
     approved ones to Stage 5 (PLAN.md protocol item 3).
5. Collect skip proposals from all reports and present them to the user
   in one batch (N21). Do not add an unapproved skip.
6. Remove the worktree and branch once the commits are on 06:
   `git worktree remove ../juju-4-pg-<N>` and
   `git branch -D pg-work/<N>-<slug>`.
7. Push only with the user's approval, using the atomic push in PLAN.md.

### After both batches

1. Run the full
   [re-sweep selection](../stages/06-compatibility.md#postgresql-re-sweep)
   once per backend at the final 09 tip, with the server to yourself,
   plus `domain/schema/testing`. Record new dated results; keep the
   earlier 2026-11-10 results. Do not add deferred packages implicitly.
2. Reassess bootstrap run 4 with the user (N22): what still fails, the
   D56 database recreation, and the cleanup targets, which need explicit
   approval before any teardown.

## Worker protocol

### Read first

- [PLAN.md](../PLAN.md) "Session protocol" and "Environment and safety"
  (the stack workflow does not apply to workers);
  [OPERATIONS.md](../OPERATIONS.md) "PostgreSQL test runs", "Real
  dqlite test runs", "Bounded runs and check-ins" and "Snap Go output
  redirection";
- [06-compatibility.md](../stages/06-compatibility.md) "Bounded repair
  tasks" and the [re-sweep](../stages/06-compatibility.md#postgresql-re-sweep);
- the [DIALECT.md](../DIALECT.md) entries your handoff cites, and D55
  in [DECISIONS.md](../DECISIONS.md);
- the code repository's root `AGENTS.md`, `AGENTS.architecture-rules.md`,
  `AGENTS.core-domain-rules.md` and any nested `AGENTS.md` above the
  files you touch.

### Workspace

- Work only in the worktree and branch the integrator gave you. Confirm
  that `HEAD` is the stated base and the tree is clean.
- Never edit the plan repository or switch any `pg-poc/*` branch. Never
  push.
- Keep logs and your report in `~/pg-poc-work/<N>/`, outside both
  repositories.
- Never print the database URL. Read it from
  `~/.config/juju-pg-poc/db-url` into the environment.

### Scope

- Change only test files, including package-local test helpers, in the
  packages your handoff lists. Do not change production code, schema
  files or the shared harness (`internal/database/testing`,
  `internal/database/postgresql/testing`, `domain/schema/testing`). If a
  repair needs one of those, stop that item and report it.
- A failure that newly appears after a fix is in scope: classify it,
  fix it if it is test-only, and keep going.
- Stop an item and report it, without changing it, when:
  - production code or schema is wrong, or production relies on what
    the test assumed (for example, row order);
  - a shared helper change would affect other packages and the impact is
    unclear;
  - the only way to pass would be a skip, or a weaker assertion.

  Leave that test failing, record the evidence and the proposed fix or
  skip (with its N10 reason) in the report, and continue with other
  items. Skips other than those your handoff lists as approved need the
  user's approval (N21).

### Repair rules

- **Bind types (X23, X25), D55.** Before you change a fixture's bind,
  check the column's declared type against production writers and
  readers. If the declared type is right, bind the matching Go type:
  `true`/`false` for `BOOLEAN`, strings for `TEXT`, `time.Time` for
  timestamps. If the declaration looks wrong, report it; do not match
  the fixture to it.
- **Timestamps (X27).** PostgreSQL `timestamptz` holds microseconds.
  Create fixture times that are microsecond-representable, for example
  `time.Now().UTC().Truncate(time.Microsecond)` where the fixture is
  made, or fixed literal times. Precedent:
  `domain/unitstate/state/commithook_test.go` (upstream `5672d2f2c3`).
  Do not truncate both sides of a comparison, and do not relax boundary
  or ordering assertions. If the time comes from production code that
  the test cannot control, report it.
- **Order (X17).** Before you make an assertion order-independent, read
  the production query and its callers. If callers treat the result as
  a set, use `tc.SameContents` or an equivalent that keeps multiplicity.
  If production depends on the order, report it.
- **Constraint errors and error text (X20, X26).** These are test-only
  (N20). Assert the constraint class with the
  `internal/database` helpers (`IsErrConstraintUnique`,
  `IsErrConstraintForeignKey`, `IsErrConstraintCheck`, and so on), or
  match the message itself in a form that holds on both backends. Do not
  match SQLite or PostgreSQL wrappers, and do not loosen to "any error".
- **Triggers.** Create abort-style triggers with
  `schematesting.FailingTriggerSQL` and drop them with
  `DropFailingTriggerSQL`; drop schema triggers with `DropTriggerSQL`.
  Execute trigger DDL through `database/sql` (`s.DB().ExecContext`, or
  `tx.ExecContext` inside `TxnRunner().StdTxn`), never sqlair: the
  PostgreSQL form is a multi-statement `$fn$` body, and sqlair parses `$`
  as an input expression. Confirm that the failing operation, its
  condition and the rollback checks still run.
- **Test DDL.** Select schema DDL with
  `ControllerDDLForBackend(BackendForTest())` or `ModelDDLForBackend`
  rather than the dqlite `schema.ControllerDDL()` / `ModelDDL()`. Write
  ad hoc test tables in SQL valid on both backends (`TIMESTAMP`, not
  `DATETIME`; no `AUTOINCREMENT`).
- Transaction closures: do not append to or assign captured variables
  inside a txn closure; reset them at the start (txncheck).

### Running tests

For PostgreSQL, assign explicit `-p` budgets whose sum across concurrent
runs is at most 8 (N19); the example below uses 4. SQLite/dqlite runs use
normal tool defaults. Run tests in the worktree, bounded
as in OPERATIONS.md, with a check at least every two minutes (N13).
Use `-race -count=1`. Write each log to `~/pg-poc-work/<N>/` with `tee`.

PostgreSQL:

```bash
set -o pipefail
export JUJU_POSTGRESQL_TEST_URL="$(cat ~/.config/juju-pg-poc/db-url)"
timeout 150s go test -count=1 -race -p 4 -timeout 120s -v \
  ./domain/<x>/state/ -run '^TestFooSuite$/^Test(A|B)$' \
  2>&1 | tee ~/pg-poc-work/<N>/<x>-pg.log
```

Real dqlite (the untagged run is the SQLite fallback, not dqlite):

```bash
set -o pipefail
unset JUJU_POSTGRESQL_TEST_URL
timeout 300s make run-go-tests TEST_PACKAGES=./domain/<x>/state/ \
  TEST_ARGS='-v -race -count=1 -timeout 120s' \
  TEST_FILTER='^TestFooSuite$/^Test(A|B)$' \
  2>&1 | tee ~/pg-poc-work/<N>/<x>-dqlite.log
```

The untagged SQLite fallback is fine for quick iteration; it does not
replace the real dqlite run.

- Once per suite family, prove that PostgreSQL was selected: run one
  test with
  `JUJU_POSTGRESQL_TEST_URL='postgres://nobody@127.0.0.1:1/none?connect_timeout=2'`
  and require a connection failure.
- Before each commit: the changed tests pass on PostgreSQL and real
  dqlite.
- Before reporting: affected tests and consumers of changed fixtures
  pass on both backends, apart from reported items. Record exact
  selectors and leave broader package coverage pending until the final
  suite. Split selectors to fit the bounded-run budget.
- Do not run bootstraps, benchmarks, stress on PostgreSQL, or full
  sweeps.
- Before each commit, run on every touched Go file:

  ```bash
  gci write --section standard --section default \
    --section "Prefix(github.com/juju/juju)" <files>
  ```

### Commits

- One commit per root cause and package. Put `bug` and `latent` changes
  in their own commits, apart from `portability` changes (PLAN.md
  protocol item 6).
- Conventional commit subject, for example
  `test(removal): use microsecond-representable schedule times`; a body
  stating before and after; then these trailers:

  ```text
  Classification: portability
  Dialect: X27
  Upstream-Home: new

  Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>
  ```

  - `Classification`: `bug`, `latent` or `portability`.
  - `Dialect`: DIALECT.md IDs, or `new` (describe it in the report).
  - `Upstream-Home`: an existing `U<n>`, `#<PR>` or break-out branch
    named in your handoff or UPSTREAM-FIXES.md; `new` for a change worth
    proposing upstream with no home yet; `none` for PoC-only changes
    (backend switches, skips).

### Report

End with `~/pg-poc-work/<N>/report.md`, and return its content as your
final message:

1. Worktree, branch, base and final commit; whether the tree is clean.
2. Commits in order: hash, subject, classification, dialect IDs,
   upstream home, tests fixed.
3. Results: commands, log paths, and pass/fail/skip counts per package
   on PostgreSQL and real dqlite; the backend-selection probe.
4. Tests still failing, each with its cause and evidence.
5. Proposed skips: test, N10 reason, the PostgreSQL coverage that
   remains, and why a repair is not possible.
6. Production findings: file and line, evidence, proposed owner (Stage 5
   or upstream) and suggested fix.
7. New dialect differences, and corrections to the plan's task rows.

## Out of scope

- Deferred PostgreSQL import (D40): `domain/modelimport` (9),
  `domain/modelimport/state/model` (3), `internal/migration` (3).
- `domain/recovery/state` (11): not triaged in this round.
- 06-schema-skip (105 skipped schema tests): a separate, optional task
  once W6 is done.
- Production query fixes: Stage 5, after user review.
