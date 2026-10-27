# Juju 4 on PostgreSQL: proof-of-concept plan

Status: **active PoC; milestone contracts being refined**.

For PoC sessions, read this file and the named stage/task's references.
For upstream-only 4.1 work, start at [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md)
and use the target repository's guidance instead; this file's session,
environment and validation instructions do not apply to that work.
Current instructions supersede handoffs; open choices are not instructions.

## Goal and scope

This proof of concept has three outcomes:

1. A functioning PostgreSQL-backed controller suitable for benchmarking
   and comparative testing.
2. SQL portability findings that expose accidental dqlite dependencies
   and incomplete architectural assumptions.
3. Concrete, independently reviewable fixes for upstream Juju 4.1.

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
needs agreement.

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
are allowed for upstream-only review branches (N11), not extra PoC trees.
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
   (N11); do not switch or restack another owner's tree. PostgreSQL package
   tests and benchmarks must not overlap. Bound every run and check its
   progress at least every two minutes
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
   `portability` changes in the stage's findings. Record every SQLite /
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
| 01 | [PostgreSQL provisioning script](stages/01-provision.md) | `pg-poc/01-provision` | in review |
| 02 | [Move schema to `domain/schema/dqlite`](stages/02-schema-move.md) | `pg-poc/02-schema-move` | in review |
| 03 | [PG driver, placeholder rewriting, errors](stages/03-driver.md) | `pg-poc/03-driver` | in review |
| 04 | [PG schema generation](stages/04-schemagen.md) | `pg-poc/04-schemagen` | done, in review |
| 05 | [Query portability fixes](stages/05-queries.md) | `pg-poc/05-queries` | X37/X38 and U11 done; `domain/secret/state` X16 sites open |
| 06 | [PG test harness](stages/06-test-harness.md) | `pg-poc/06-test-harness` | D42 runtime-entry gate met; test SQL repairs open |
| 07 | [`db-backend` / `db-url` config plumbing](stages/07-config.md) | `pg-poc/07-config` | in review |
| 08 | [PG dbaccessor worker and bootstrap](stages/08-dbaccessor.md) | `pg-poc/08-dbaccessor` | in progress (bootstrap layer, 08-tracker, 08-worker, 08-upgrade-selector and 08-delete committed; 08-cleanup-tool next) |
| 09 | [Single-controller functionality and comparison](stages/09-bootstrap.md) | `pg-poc/09-bootstrap` | not started; workload open |
| 10 | [Juju controller HA, shared PG primary](stages/10-ha.md) | `pg-poc/10-ha` | not started; follows first comparison |
| 11 | [Future work](stages/11-future-work.md) | n/a | not planned |

Next: 08-cleanup-tool, the operator script
`scripts/postgresql-poc/drop-juju-schemas.py`, then the Stage 9 handover
([handoff](handoffs/2026-10-27-stage-8-cleanup-tool.md)). The top is
`08=05c37d6e9f`. On 2026-10-27 the per-test schema teardown was made to
fail a test whose CASCADE drop reaches objects outside its schema (N15)
at `06=e7f3822281`, and the stack restacked; earlier 08 hashes in the
stage files are historical. 08-delete is done
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
(N4, N8). The base is upstream `4.1` at `a8591fe1b8`. Stage 5 also
carries unchanged cherry-picks of open upstream PRs (D43; listed in
UPSTREAM-FIXES.md). The **top** is the
highest stage branch that exists, currently `pg-poc/08-dbaccessor`.

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

## Secrets

- Never commit a DB URL or password, in either repository.
- Keep the URL in an environment variable (`JUJU_PG_POC_DB_URL`) or a file
  outside the repositories with mode 0600 (the Stage 1 script writes
  `~/.config/juju-pg-poc/db-url`).
- Scripts print credentials only to stdout or a file the user names.
