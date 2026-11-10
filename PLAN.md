# Juju 4 on PostgreSQL: proof-of-concept plan

Status: **active PoC; milestone contracts being refined**.

For PoC sessions, read this file and the named stage/task's references.
For upstream-only 4.1 work, start at [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md)
and use the target repository's guidance instead; this file's session,
environment and validation instructions do not apply to that work.
Current instructions supersede handoffs; open choices are not instructions.

## Goal and scope

This proof of concept has four outcomes:

1. A functioning PostgreSQL-backed controller suitable for benchmarking
   and comparative testing.
2. SQL portability findings that expose accidental dqlite dependencies
   and incomplete architectural assumptions.
3. Concrete, independently reviewable fixes for upstream Juju 4.1.
4. A Juju 4.1-based branch that can run the domain test suite against
   PostgreSQL to maintain SQL consistency as future changes land.

Compare three cases (D38): a pinned stock upstream Juju build, the PoC
build using dqlite, and the same PoC build using PostgreSQL. Record the
source revisions and deployment configurations. Upstream fixes as they
become ready; their release value does not depend on finishing the PoC.

In scope (D1): LXD machine controllers including `enable-ha`, fresh
bootstrap only, and an external PostgreSQL reached via a URL. dqlite
remains the default and keeps working.

Out of scope: Kubernetes controllers, migrating a dqlite controller,
`dbrepl` / `dbreplaccessor`, credential rotation, the controller-charm
relation, production hardening and performance optimisation. Benchmarking
and comparative testing are in scope; workload-specific acceptance still
needs agreement. Test-suite performance work for the PostgreSQL regression
branch described below is also in scope.

### PostgreSQL SQL-regression branch

Bootstrapping remains a higher priority than this outcome. As the breakouts
in [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md) and
[UPSTREAM-PLAN.md](UPSTREAM-PLAN.md) progress, retain a path to a Juju
4.1-based branch with the shared fixes and PostgreSQL test infrastructure
needed to run the domain test suite. The goal is repeatable regression
validation of SQL consistency, independent of PostgreSQL controller runtime
support. Include the SQL-owning infrastructure and worker packages outside
`./domain/...` identified by N16; a domain-only run leaves coverage gaps.

If the suite can run in **under 15 minutes**, it can become part of the
pre-commit checks run on every PR. Measure the complete selected suite on
documented hardware and configuration before adopting that gate; passing a
smaller smoke selection does not establish this target. This is a desired
end state, not an already available check or a new acceptance requirement
for each individual upstream breakout.

### Shared SQL/schema fixes versus PostgreSQL support

The PoC's PostgreSQL runtime, driver, generated DDL and harness remain in
scope here; they are not requirements for upstream 4.1 breakouts. Those
breakouts deliver SQL/schema correctness on SQLite and real dqlite under
[the upstream acceptance contract](UPSTREAM-PLAN.md#purpose-and-sqlschema-acceptance)
(D55), without PostgreSQL-specific SQL or support infrastructure.

For shared findings, determine the intended domain type and query semantics
before choosing a fix. Correct an incorrect declaration or SQL construct
rather than merely making Go bind/scan it successfully. Keep genuine
legacy/API conversions at compatibility boundaries. A PoC workaround or
PostgreSQL pass is evidence, not proof that the shared root cause is fixed
or a mandatory upstream design. Correctly modeled text, such as U3's full
unsigned node IDs, must not be changed to signed integers just because Go
uses a numeric type.

Preserve historical PoC results and identify any remaining schema correction
separately. U21's decimal-text implementation unblocks the PoC but leaves the
integer relation-key schema correction pending in the upstream task. Do not
silently rewrite the PoC stack to mirror a revised upstream design; use the
existing ownership and reconciliation procedure. Ask when a task's prescribed
approach conflicts with the root-cause goal rather than widening its scope.

## Milestones

| Milestone | Completion | Main stages |
|---|---|---|
| Runtime integrated | Bootstrap and namespace/worker lifecycle work; required compatibility checks pass. | 5–8 |
| Functional single controller | End-to-end lifecycle passes on PoC dqlite and PostgreSQL, including the dedicated cross-model secrets fix (D40). | 5-CMR, 6, 9 |
| First comparison | Run the agreed workload on stock upstream, PoC dqlite and PoC PostgreSQL; record correctness and performance evidence. | 9, [comparison contract](BENCHMARKS.md) |
| Juju controller HA | Three Juju controllers share one writable PostgreSQL primary; controller loss/rejoin preserves operation. | 10 |

The first comparison precedes HA (D39). Read-replica routing and PostgreSQL
primary failover experiments are not part of Stage 10. PostgreSQL model
import/export is deferred (D40); shared fixes must still preserve upstream
import/export compatibility.

Portability findings and upstream 4.1 fixes are continuing deliverables,
not a final stage after HA. Record unfixed findings as well as fixes.
D42 defines the Stage 8 entry gate separately from functional acceptance.
Benchmark workloads and unresolved worker contracts still require approval
before their corresponding execution tasks begin.

## Context map

| Read | When |
|---|---|
| This file + named [stage](stages/) | Every session; current task, branch and stop conditions. |
| [DECISIONS.md](DECISIONS.md) | Only cited D*/N* entries; before proposing a change to them. |
| [DIALECT.md](DIALECT.md) | Relevant X* entries when changing SQL or diagnosing backend differences. |
| [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md) | Evolving findings, provenance, upstream and PoC reconciliation status. |
| [UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md) | Integration agent: reusable procedure for bringing landed fixes into the PoC. |
| [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md) | Standalone stock-4.1 tasks; upstream agents do not need this PoC plan. |
| [OPERATIONS.md](OPERATIONS.md) | Building, testing, regenerating or operating the PoC. |
| [Stage 6 task/results detail](stages/06-compatibility.md) | Selected compatibility task or validation run. |
| [BENCHMARKS.md](BENCHMARKS.md) | Designing or executing the three-way comparison. |
| [Cross-model secrets](investigations/crossmodel-secrets.md) | U11 design/fix task (done; cherry-picked under D43). |
| [ANALYSIS.md](ANALYSIS.md), [history/](history/), [handoffs/](handoffs/) | Exploration rationale or historical evidence only; not prerequisite reading unless a task cites a section. |

## Repositories

Sessions run in `~/dev/go/src/github.com/juju/juju-4-postgresql`
(`origin=jameinel/juju`). Keep PoC work in one tree. Dedicated worktrees
are allowed for upstream-only review branches (N11) and for Stage 6
test-repair workers (N18), not extra PoC trees.
Plans live in sibling `../juju-4-postgresql-plan`
(`origin=jameinel/juju-4-postgresql-plan`, branch `main`); grant session
access if needed. Never commit plan documents to the code repository.

## Starting a session

Prompt: "Read ../juju-4-postgresql-plan/PLAN.md and run task <id> in
Stage <N>. Use <controller> to host PostgreSQL if needed. Stop after that
task."

A dispatchable task needs a branch, prerequisites, code entry points,
approved behaviour, bounded changes, checks and stop conditions. Clarify
missing inputs; do not ask an executor to design and implement a backlog.

## Session protocol

1. Read this plan, the named task and its required references, plus the
   code repository's root and applicable nested `AGENTS.md` guidance.
2. Check both repositories' status and branch tips. Do not overwrite
   another session's changes or infer current state from an old commit log.
3. Work on one task. Route necessary earlier-stage fixes to their owning
   branch; stop for new scope or design choices rather than expanding the
   task. Do not weaken assertions or add skips merely to get a green run.
4. Confirm open choices with the user before implementation; record only
   confirmed decisions in DECISIONS.md. Distinguish a proposed fix from an
   approved one, and implemented work from verified work.
5. Keep one PoC implementation/restack stream. Upstream-only fixes may
   run concurrently in dedicated worktrees under [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md)
   (N11), and Stage 6 test-repair workers in theirs (N18); do not switch
   or restack another owner's tree. Concurrent PostgreSQL test runs may
   share the server while their `-p` values sum to at most 8; bootstrap,
   benchmarks, stress and full sweeps need it alone (N19). Run multiple
   packages from one `go test` invocation with `-p 4` to `-p 8`; tests
   within a package run
   serially, so packages one at a time are several times slower
   ([OPERATIONS.md](OPERATIONS.md#postgresql-test-runs)). Bound every run
   and check its progress at least every two minutes
   ([OPERATIONS.md](OPERATIONS.md#bounded-runs-and-check-ins), N13).
6. Classify every change made to get PostgreSQL working, whether in code,
   tests or schema:
   - `bug`: the dqlite behaviour is wrong today.
   - `latent`: dqlite is right today only because of SQLite leniency (type
     affinity, coercion, double-quoted strings) or a test that ignores an
     error.
   - `portability`: valid on dqlite and only invalid on PostgreSQL.

   Put each `bug` and `latent` change in its own commit, apart from
   PoC-only changes, so it can be proposed upstream on its own. Add it to
   UPSTREAM-FIXES.md, together with defects found but not fixed. Record
   `portability` changes in the stage's findings and register independently
   useful upstream candidates in UPSTREAM-FIXES.md. A portability
   classification does not exclude a shared SQL improvement from upstream,
   or make PostgreSQL-only support appropriate there. Record every SQLite /
   PostgreSQL difference behind a change, or found and not yet handled,
   in DIALECT.md, including driver workarounds.
7. Before ending an implementation session:
   - Update current task status, blockers and revision-specific evidence,
     and this status table. Fold actionable handoff items into current
     documents; history must not be the only place the next task is defined.
   - Keep useful exploration history in a linked reference, not the next
     task's mandatory context. Avoid duplicating commit inventories.
   - Commit and push the plan repository and code branches under the stack
     workflow. Documentation/design sessions do not restack code.

## Status

| # | Stage | Branch | Status |
|---|---|---|---|
| - | Plan documents | plan repository `main` | in review |
| 00 | [Base fixes](stages/00-base-fixes.md) | `pg-poc/00-base-fixes` | done, in review |
| 01 | [PostgreSQL provisioning script](stages/01-provision.md) | `pg-poc/01-provision` | in review (also carries the D52 cleanup tool) |
| 02 | [Move schema to `domain/schema/dqlite`](stages/02-schema-move.md) | `pg-poc/02-schema-move` | in review |
| 03 | [PG driver, placeholder rewriting, errors](stages/03-driver.md) | `pg-poc/03-driver` | in review |
| 04 | [PG schema generation](stages/04-schemagen.md) | `pg-poc/04-schemagen` | done, in review; 04-current-timestamp (D56) implemented and restacked |
| 05 | [Query portability fixes](stages/05-queries.md) | `pg-poc/05-queries` | X37/X38, U11, 05-SECRET-X16, 05-BOOT-FIXES, 05-X16-rest, 05-unitstate-relation-key and 05-lease-db-time (D16) done; U15/U16 cherry-picked (D43); no other known production query defect open |
| 06 | [PG test harness](stages/06-test-harness.md) | `pg-poc/06-test-harness` | D42 runtime-entry gate met; trigger drop helpers done locally (`f4d3cbb673`); W1–W4 dispatched (N18), W5–W6 pending; 146-package re-sweep baseline still has 35 failing packages |
| 07 | [`db-backend` / `db-url` config plumbing](stages/07-config.md) | `pg-poc/07-config` | in review |
| 08 | [PG dbaccessor worker and bootstrap](stages/08-dbaccessor.md) | `pg-poc/08-dbaccessor` | implementation tasks done (bootstrap layer, 08-tracker, 08-worker, 08-upgrade-selector, 08-delete, 08-cleanup-tool and 08-delete-locks committed); per-process test lock key added (N17) |
| 09 | [Single-controller functionality and comparison](stages/09-bootstrap.md) | `pg-poc/09-bootstrap` (same commit as 08) | 09-functional scenario agreed (D54); query blockers from runs 2 and 3 and the lease-clock regression are fixed; run 4 deferred until the Stage 6 repairs and a new re-sweep (N22), then D56 database recreation |
| 10 | [Juju controller HA, shared PG primary](stages/10-ha.md) | `pg-poc/10-ha` | not started; follows first comparison |
| 11 | [Future work](stages/11-future-work.md) | n/a | not planned |

Next:

1. Run the [Stage 6 parallel repairs](handoffs/2026-11-10-stage-6-parallel-repairs.md)
   (N18): Step 0 is verified at `09=0bb336f089`; W1–W4 are running on
   GPT-6.1 Terra. Dispatch W5–W6 as slots open, at most four at a time, and
   integrates each onto Stage 6. Workers report production defects
   rather than fixing them; route approved ones to Stage 5. Then rerun
   the re-sweep. Out of scope for this round: the three deferred import
   packages (D40), `domain/recovery/state` and 06-schema-skip; see the
   handoff.
2. Reassess bootstrap run 4 with the user (N22). Before bootstrap,
   inspect resources and obtain explicit approval of cleanup targets
   before any teardown or database recreation: 04-current-timestamp
   (D56) changes the schema hash. Then run bootstrap 4 and the D54
   scenario, then the dqlite comparison run.
3. Dispatch the Stage 5 upstream break-outs scoped in
   [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md#further-break-outs-from-poc-stage-5-2026-11-05),
   respecting current claims and the D55 review. Storage #23766 has a
   user-requested merge hold pending attachment-invariant investigation and
   rework, not an instruction to merge the deterministic-row version.
   U21's integer-schema rework is published as #23769; grouped-column work
   includes CA-certificate preservation. `4.1-boolean-and-aggregate-literals`
   remains separately scoped. Alongside these, audit the `0` / `1` seed
   rows written to `BOOLEAN` columns (UPSTREAM-PLAN.md D55 audit table).

Stage 5 fixes are validated on PostgreSQL at the 09 tip, not at 05:
`JUJU_POSTGRESQL_TEST_URL` has no effect before the Stage 6 harness.

Current local tips (2026-11-10, base `5259edff43`):
`00=3133b016cc`, `01=8edc93c531`, `02=e7b50aad4f`, `03=92bf73a157`,
`04=f957a3ae5e`, `05=76aac853dd`, `06=f4d3cbb673`, `07=de24e7d3c2`,
`08=09=0bb336f089`. Step 0 of the parallel repairs is committed locally
and verified on PostgreSQL and real dqlite; W1–W4 branch from this 09
tip. See [the repair run](stages/06-compatibility.md#parallel-repair-run-2026-11-10-in-progress)
for evidence. No repair-round commits have been pushed.

Previous local tips (2026-11-09, base `5259edff43`):
`00=3133b016cc`, `01=8edc93c531`, `02=e7b50aad4f`, `03=92bf73a157`,
`04=f957a3ae5e`, `05=76aac853dd`, `06=ebba92514b`, `07=fae300116a`,
`08=09=a3818782aa`. D56 and the D16 lease-clock correction are restacked
through Stage 9. At `09=a3818782aa`, `go build ./...` and both affected
lease packages passed with `-race` on SQLite, real dqlite and PostgreSQL;
an invalid-URL probe proved both database suites selected PostgreSQL.
The 146-package PostgreSQL re-sweep completed in 21m50s: 108 packages
passed, 35 failed and three had no tests; 170 leaf tests were skipped.
See [Stage 6](stages/06-compatibility.md#postgresql-re-sweep) for the
failure inventory and coverage gaps. This is not a passed readiness gate.
At the previous `09=0aed8123db`,
`go build ./...` passed; the focused charm
ordering tests passed with `-race` on PostgreSQL and dqlite, the
controller/model PostgreSQL schema comparison passed, and the legacy
import delta test passed on dqlite. An invalid-URL probe confirmed that
the state suite selected PostgreSQL. See [Stage 4](stages/04-schemagen.md)
for the earlier affected package results and remaining recreation
requirement, and [Stage 5](stages/05-queries.md#05-lease-db-time-lease-times-from-the-database-clock-done)
for the lease validation and outstanding checker findings.

The published tips before the lease restack were `04=f957a3ae5e`,
`05=efc7f1d195`, `06=b17e623484`, `07=b785b0b27d`,
`08=09=0aed8123db`. Before the D56 restack on 2026-11-09 they were
`04=53a0bf03d9`, `05=640d3bcabf`, `06=4b1a462df0`, `07=008f3fcf15`,
`08=09=760f3b675f`. An earlier set was `05=3a8f0090c2`,
`06=add66191aa`, `07=d700353eb0`, `08=09=eac5b916c7`. Previously verified
on PostgreSQL (server `lxd36-pgtest`,
disposable profile, then unit-test profile): the D42 entry packages,
sweep 2 over 139 packages at `08=ba634a0f13` (37 failing, classified in
[Stage 6](stages/06-compatibility.md#postgresql-sweep-2)), the secret
and unitstate re-run at `05=3a8f0090c2`, the N17 benchmarks at
`08=eac5b916c7`, and the 05-X16-rest and 05-unitstate-relation-key
packages at `09=760f3b675f`. Bootstrap runs 2 and 3 are in
[the run log](stages/09-bootstrap.md#09-functional-run-log).

On 2026-11-05 the stack was rebased from the bottom onto upstream 4.1
`5259edff43` (was `a8591fe1b8`), dropping the commits that landed
upstream ([reconciliation record](UPSTREAM-FIXES.md#poc-reconciliation-onto-5259edff43-2026-11-05)).
The PostgreSQL DDL was regenerated at Stage 4, and U15 (juju/juju#23710,
#23723) and U16 (juju/juju#23728) were cherry-picked onto Stage 5 under
D43 with a second regeneration. The published tips from that rebase were
`00=3133b016cc`, `01=bd7b6cb0f0`, `02=8f1c3532d6`, `03=e13f3d1ba6`,
`04=4248ede57c`, `05=52b6ba047c`, `06=a7dc3099c3`, `07=b0829f3cca`,
`08=98847cacb4` (verified on the remote on 2026-11-05).
Old tips are tagged `pg-poc-base-a8591fe1b8/<stage>`.
New upstream-only branches base on `5259edff43` or later 4.1.

Historical (before the rebase): 09-functional run 1 failed on U15
(nullable primary key for the wildcard exposed endpoint, X44) and U16
(`RegisterAgentBinary` after a conflict, X18); see
[the run log](stages/09-bootstrap.md#09-functional-run-log). U15 is split
into two upstream reviews: `4.1-nullable-primary-keys` keeps the charm-key
fix in juju/juju#23710; `4.1-application-exposure-scope` is stacked on it
and owns the wildcard exposure fix and combined primary-key guard. The
split and second PR are committed locally and refreshed onto upstream
`8369e6e2a9` (`charm=ba21f48ad3`, `exposure=f31de64e52`), published as
juju/juju#23710 and juju/juju#23723 after explicit approval; both are
now cherry-picked onto Stage 5 (above). See [the review scope](UPSTREAM-PLAN.md#nullable-primary-key-review-split).
U17 (65 nullable INT lookup keys) is planned as an independent follow-up
session; it does not need either U15 review to land first.
Both fixes were needed before the next PostgreSQL bootstrap; the charm
PR alone does not fix wildcard exposure. A proposed
[same-database guard](stages/09-bootstrap.md#09-same-db-guard-stop-two-controllers-sharing-one-database-proposed)
is recorded and not scheduled. The top before the rebase was
`08=950bba743e`; hashes below are historical. 08-delete-locks
is done ([result](stages/08-dbaccessor.md#08-delete-locks-result)):
every PostgreSQL schema change transaction (`Schema.Ensure`,
`DeleteSchema`) first runs the dialect's `BeginSchemaChangeSQL`, which
sets READ COMMITTED and takes a per-database advisory lock (D53). A
model delete takes about 4,400 locks and a create about 3,600, against
a shared table of 8,768 (X42), so they now run one at a time. READ
COMMITTED is a D7 exception: a long SERIALIZABLE `Ensure` exhausted the
predicate lock table under load (X43). Stage 11 records an experiment
to run all transactions at READ COMMITTED. 08-cleanup-tool is done
([result](stages/08-dbaccessor.md#08-cleanup-tool-result)):
`scripts/postgresql-poc/drop-juju-schemas.py` on Stage 1
(`01=bd0f4e7638`, D52) lists and drops PoC controller, model and test
schemas, one transaction per schema. Stages 02–08 were restacked onto
it; earlier 08 hashes in the stage files are historical. On 2026-10-27
the per-test schema teardown was made to
fail a test whose CASCADE drop reaches objects outside its schema (N15)
at `06=e7f3822281`, and the stack restacked. 08-delete is done
([result](stages/08-dbaccessor.md#08-delete-result)): `pgWorker.DeleteDB`
drops a model's schema with RESTRICT (D47), in a READ COMMITTED
transaction, an approved exception to D7 (D47 amendment, X40).
On 2026-10-27 the PostgreSQL test session timeouts (N14) were added at
`06=4615e24467` and the stack restacked.
08-upgrade-selector is done at `08=306db7518f`
([result](stages/08-dbaccessor.md#08-upgrade-selector-result)): the
upgrade worker selects its controller and model DDL from the same
`DBConnectionSource` as dbaccessor (D51). 08-worker is done
at `08=aa09e41411`
([result](stages/08-dbaccessor.md#08-worker-result)): `pgWorker`, manifold
selection from `DBConnectionSource` (D49) and a fatal loss of the
controller schema (D50). 08-tracker is done
at `08=45f3bdf358`
([result](stages/08-dbaccessor.md#08-tracker-result)): D48 tracker
interfaces with `PostgresNamespaces`, and the U14 pool-leak fix, which
also applies to stock 4.1. The D42
runtime-entry gate is met at `08=c5f185e6f6`
([evidence](stages/06-compatibility.md#d42-runtime-entry-gate-run)):
all 13 entry packages pass on both backends; status (48) and removal (47)
failures are all recorded test-side tasks. `domain/schema` still skips its
trigger and schema tests on PostgreSQL (06-schema-skip).
X37/X38 and U11 are done; U11 is cherry-picked from juju/juju#23688,
which is still open. [Stage 5](stages/05-queries.md) keeps the
`domain/secret/state` bare `GROUP BY` sites (X16) found while verifying
U11. [Stage 6](stages/06-test-harness.md) owns the
remaining test repairs and the functional gate. Do not repeat the old
bootstrap-query fixes. Broader portability findings still gate
functionality rather than disappearing from scope.

## Stack workflow

Every stage branch sits on the one below it in the table, in one line
(N4, N8). The base is upstream `4.1` at `5259edff43` (since 2026-11-05;
previously `a8591fe1b8`, old tips tagged `pg-poc-base-a8591fe1b8/*`).
Stage 5 also
carries unchanged cherry-picks of open upstream PRs (D43; listed in
UPSTREAM-FIXES.md). The **top** is the
highest stage branch that exists, currently `pg-poc/09-bootstrap` (at the
same commit as `pg-poc/08-dbaccessor`).

- **New stage**: create its branch from the top
  (`git switch -c pg-poc/NN-<slug> <top>`).
- **Fix that belongs to an earlier stage**:
  1. `git switch pg-poc/NN-<slug>` and commit the fix there.
  2. `git switch <top> && git rebase --update-refs pg-poc/NN-<slug>`. This
     replays the later stages on the fix and moves every branch between.
- Amendments and upstream-base changes: use
  [the exceptional procedures](OPERATIONS.md#exceptional-stack-operations).
  Change the upstream base only when the user asks.
- After any restack, run `go build ./...` on the top and the tests of the
  packages the fix touched.
- **Push** every stage branch together:

  ```sh
  git push --atomic --force-with-lease origin \
    $(git for-each-ref --format='%(refname:short)' 'refs/heads/pg-poc/*')
  ```

- Commit or stash before switching branches. GoLand re-indexes only the
  files that change, so switching between adjacent stages is cheap.
- To read another stage without switching, use
  `git show pg-poc/NN-<slug>:<path>` or `git diff pg-poc/NN-<slug> -- <path>`.
- `pg-poc/00-base-fixes` holds only fixes for pre-existing problems that
  are unrelated to the PoC. Stage 2 stays purely mechanical so it can be
  reviewed, and possibly proposed upstream, on its own.

## Environment and safety

- The default `juju` is Juju 3.6, used to host PostgreSQL. Invoke the PoC
  client explicitly as `$(go env GOPATH)/bin/juju`; build from the code
  tree. A reference snap is not validation of modified dqlite code.
- Pass the hosting controller as a parameter (`PG_HOST_CONTROLLER`), not
  a hardcoded name. Clients share local Juju state: target controller and
  model explicitly, especially for teardown or fault injection.
- Test and cleanup operations must not touch unrelated schemas, models or
  controllers. Confirm destructive targets; never clean up the PG host as
  part of a Juju controller-failure test.
- Read [OPERATIONS.md](OPERATIONS.md) for build/test commands, backend
  proof, timeouts, schema regeneration, tool locations and server limits.
- `go` is a snap. Never redirect its output straight to a file
  (`go test ... > log 2>&1`): AppArmor silently discards it, and the exit
  code is unaffected, so nothing signals the loss. Pipe instead, for example
  `set -o pipefail; go test ... 2>&1 | tee "$LOG"`, and check that the log
  is growing at the first check-in
  ([OPERATIONS.md](OPERATIONS.md#snap-go-output-redirection)).

## Secrets

- Never commit a DB URL or password, in either repository.
- Keep the URL in an environment variable (`JUJU_PG_POC_DB_URL`) or a file
  outside the repositories with mode 0600 (the Stage 1 script writes
  `~/.config/juju-pg-poc/db-url`).
- Scripts print credentials only to stdout or a file the user names.
