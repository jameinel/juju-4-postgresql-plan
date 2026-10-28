# Stage 09: functional single controller and first comparison

Branch `pg-poc/09-bootstrap`, to be created on `pg-poc/08-dbaccessor`.
Status: 09-functional started 2026-10-28 (scenario fixed in D54); first PostgreSQL bootstrap fails, see the run log below. Read D38–D40 and [BENCHMARKS.md](../BENCHMARKS.md).

## Prerequisites

Stage 8 runtime tasks and required Stage 6 coverage pass at a recorded
revision. The dedicated U11 cross-model secrets task must be implemented
and verified before the functional milestone. PostgreSQL model import/export
is deferred, not an implicit smoke-test requirement.

## 09-functional: define, then run the correctness scenario

Before dispatch, agree a pinned charm/workload, readiness/completion
signals, bounded waits, controller/model names and cleanup. Include:

1. Build with `make juju jujuagentd` at the recorded revision. Bootstrap
   with the explicit built client and `--build-agent`. Supply
   `db-backend=postgresql`, `db-url` and any agreed `db-ca-cert` through a
   private bootstrap config file outside both repositories (mode 0600),
   rather than putting credentials in recorded commands or process args.
2. Prove controller/model persistence is in PostgreSQL, not just that a
   bootstrap command exited successfully.
3. Add a model, deploy and exercise a real operation, observe convergence
   through status/watchers, and check a controller restart reopens the
   existing namespaces.
4. Remove the application and destroy the added model; wait for completed
   teardown and verify only its schema is dropped.
5. Validate the agreed cross-model relation/secrets scenario, informed by
   U11's approved multi-unit semantics. Do not call the PoC functional
   while that fix is still deferred.
6. Repeat correctness checks with default dqlite from the same PoC source
   revision, using real built agent binaries. `juju_4` is optional reference
   evidence, not a substitute for this regression test.

Record failures, skips, revisions and redacted logs. Do not assume an empty
error log or successful `status` proves all operations above completed.
New production failures return to the owning stage; no opportunistic
scope expansion in the smoke-test task.

## 09-compare: three cases

After correctness, agree and run the experiment in BENCHMARKS.md:
stock upstream, PoC dqlite and PoC PostgreSQL. Pin the upstream revision
and both PoC builds. Record total deployment resources and material
configuration differences, not just controller process usage.

Acceptance is reproducible, interpretable measurements with correctness
preserved, not a promised speedup. Start with single controllers; Stage 10
follows. Workload, scale, repetitions and TLS comparison remain open and
must be fixed before assigning the run.

## 09-functional run log

### Run 1: PostgreSQL bootstrap (2026-10-28, `09=950bba743e`, no code changes)

`make juju jujuagentd`, then `juju bootstrap localhost pgpoc-pg
--config <0600 file> --build-agent`. The controller machine started and
created both schemas (`juju_controller` with 61 patches, the controller
model with 156), so the runtime path from Stage 8 works. Bootstrap never
completed; the `bootstrap` worker failed repeatedly:

1. First attempt: `ensuring controller application: exposing controller
   application: inserting exposed endpoints to CIDRs: null value in column
   "application_endpoint_uuid" ... (23502)`. U15 / X44: a nullable primary
   key column used for the wildcard endpoint. Not fixed; needs a schema
   design choice.
2. Every retry: `saving agent binary metadata ... current transaction is
   aborted (25P02)`. U16 / X18: `RegisterAgentBinary` queries after a
   primary key conflict. A consequence of the first failure's retry, but a
   real portability defect. Not fixed.

Also seen, probably secondary: `api-address-setter: getting network for
controller unit "controller/0": unit not found`.

Cleanup: the container was deleted with `lxc delete -f` (`kill-controller`
hung on the unreachable API), the client entry removed with `juju
unregister`, and both schemas dropped with `drop-juju-schemas.py
--controller`. The database was left with no Juju schemas.

Next: the user wants U15 split out as an upstream fix (NULL in a primary
key is a bad design; see `4.1-nullable-primary-keys` in UPSTREAM-PLAN.md),
with U16 beside it. Once both exist, cherry-pick them under D43 onto their
owning stage (or apply a PoC-local workaround meanwhile), restack, and re-run. Expect further
bootstrap findings; no dqlite run has been made yet.
