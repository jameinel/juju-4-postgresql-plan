# Stage 09: functional single controller and first comparison

Branch `pg-poc/09-bootstrap`, on `pg-poc/08-dbaccessor`.
Status: 09-functional started 2026-10-28 (scenario fixed in D54). Run 1
failed on U15/U16 (integrated). Runs 2 and 3 (2026-11-05) each found a
PostgreSQL-only query defect on the login and uniter paths (fixed on
Stage 5). Run 4 at `09=75223c9543` created the PostgreSQL schemas but
was blocked by the Charmhub outage. Its instance and schemas have been
cleaned up with user approval. Run 5 at `09=f3340c6710` successfully
bootstrapped `pgpoc-pg`, verified PostgreSQL persistence and reopened the
same namespaces after restarting the controller agent. The controller
remains running. D54's PostgreSQL workload, multi-unit secrets and restart
checks passed. The same-revision dqlite secret/restart checks and mixed CMR
checks in both directions also passed, but the original dqlite consumer's
`dummy-sink/1` remains idle with "Waiting for token". The complete dqlite
workload is therefore not verified healthy. Direct database inspection
found the token present and both sinks recording the current settings
version, despite unit 1's last hook reading an empty token. The underlying
delivery/read inconsistency remains unresolved. All resources are preserved
at the user's request; no teardown or schema-removal isolation check ran.
The next evaluation is [09-token-resume](#09-token-resume-evaluate-the-hooksettings-version-mismatch).
See the run log below. Read D38–D40 and [BENCHMARKS.md](../BENCHMARKS.md).

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

## 09-token-resume: evaluate the hook/settings-version mismatch

Status: investigation open; next steps recorded, not a chosen fix or
authorization to change the preserved deployments. Controller debugging
is deferred at the user's request. The observations below apply to
`pg-poc/09-bootstrap` at `f3340c67107829347ee750f7e40e806e563411e9`.
No production code was changed during the evaluation or investigation.

The acceptance signal is the token visible to every expected consumer,
not only an active remote endpoint, an idle agent or an acknowledged
settings version. Current database snapshots cannot reconstruct the
settings or cache contents at the time of the empty hook read.

Next steps, in order:

1. Preserve the original models and their evidence. Continue with static
   inspection and read-only database/log queries; do not repair unit 1 by
   setting another token, rerunning its hook, restarting an agent or
   cycling its relation. Use the database queries below to compare the
   source databag, its settings hash and both sinks' persisted versions.
   Record snapshots with UTC timestamps and distinguish current state from
   the 00:38:10 UTC hook-time observations.
2. Trace the version and settings paths independently, then identify where
   they can diverge during resume. Start at
   `domain/relation/state/{relation.go,related_unit_watcher.go}` and
   `apiserver/facades/agent/uniter/watcherrelationunit.go` for watcher
   selection and conversion of settings hashes into versions. Follow
   `internal/worker/uniter/remotestate/watcher.go`,
   `relation/{resolver.go,state.go}` and
   `runner/context/{contextfactory.go,cache.go}` under that uniter directory
   for hook selection, version persistence and remote-settings cache
   invalidation. Evaluate coalesced resume/settings events, version changes
   between hook selection and execution, and interleaved action contexts.
   These are hypotheses, not established causes.
3. Before any live reproduction, obtain approval for separate disposable
   offer/consume models on each controller. Pin the same source revision
   and charms as D54, with two consumers. Begin with one suspend/resume
   cycle per backend and no concurrent `juju exec` calls; observe through
   status, logs and read-only database queries. If both pass, run a separate
   case with the original D54 action/secret-read interleaving. Bound each
   convergence wait at 600 seconds. Stop at the first mismatch and preserve
   the new models too; cleanup needs separate approval.
4. For a failing case, correlate the version selected for the hook, whether
   its settings read used a cache, the value returned, the version persisted
   after completion, and the next watcher notification. Use existing scoped
   logging first. Any additional instrumentation is a separately reviewed
   change, not permission to attach a debugger or modify the live agents.
   Build a deterministic regression in the owning watcher/resolver/cache
   package once this distinguishes a missed event from stale read data or
   incorrect version acknowledgment. Cover the relevant event ordering
   and action interleaving; do not infer a dqlite-only defect from one run.
5. Before another D54 acceptance run, correct the evaluator's readiness
   check: require every expected sink unit to be present, active/idle and
   reporting `Token is foobar`. Demonstrate that one waiting sink, a wrong
   token or a missing unit fails the check. After a separately approved fix,
   repeat the affected scenarios on both backends and both mixed-CMR
   directions, including the secret/restart checks. Use targeted `-race`
   tests and bounded stress for changed concurrent code; validate real
   dqlite and PostgreSQL if the persistence layer changes.

Stop conditions: no speculative production fix or new controller debugging.
Full D54 acceptance remains open until both consumers converge correctly
and separately authorized teardown/schema-isolation checks are completed.
Publishing the research or code stack does not authorize these operations.

## 09-compare: three cases

After correctness, agree and run the experiment in BENCHMARKS.md:
stock upstream, PoC dqlite and PoC PostgreSQL. Pin the upstream revision
and both PoC builds. Record total deployment resources and material
configuration differences, not just controller process usage.

Acceptance is reproducible, interpretable measurements with correctness
preserved, not a promised speedup. Start with single controllers; Stage 10
follows. Workload, scale, repetitions and TLS comparison remain open and
must be fixed before assigning the run.

## 09-same-db-guard: stop two controllers sharing one database (proposed)

Status: proposed, not approved or scheduled. Raised by the user on
2026-10-28 after a manual bootstrap ran alongside run 1: one database per
controller is accepted (D3), but a second bootstrap against a database that
already holds a controller should fail clearly, so users cannot damage
themselves. Whether run 1 was in fact disturbed by that manual bootstrap is
unknown; its failures (U15/U16) are deterministic and do not need it.

What exists (read from the code, not yet exercised live):

- `BootstrapPostgres` (`internal/database/bootstrap_postgresql.go`) creates
  `juju_controller` and `juju_<controller model uuid>` in one transaction and
  fails with `ErrSchemaExists` ("bootstrap requires a database with no Juju
  state") if either exists. A concurrent second bootstrap blocks on the
  first's uncommitted schema and then fails the same way.
- It runs on the controller machine in bootstrap-state, so the user finds out
  only after the instance is provisioned and the agent installed, and only
  through the bootstrap output.
- The model schema is keyed by the new controller model UUID, so a second
  bootstrap collides only on `juju_controller`.

Gaps to check and decide:

1. Live check: bootstrap a second controller at a database that holds a
   (failed or complete) first one. Record the client output, what is left on
   the machine/LXD, and whether the first controller is untouched.
2. Early failure: a client-side pre-flight at `juju bootstrap`. The client may
   not reach the database, so it can only be best effort (fail if it can
   connect and finds `juju_controller`; otherwise warn and go on). It would
   link the PostgreSQL driver into the client. Open choice.
3. Runtime ownership: nothing at runtime checks that `juju_controller` belongs
   to this agent's controller. The recorded controller UUID is in the schema
   (the cleanup tool reads it). A controller pointed at another controller's
   database, for example by a mistaken restore or `enable-ha` configuration,
   would open it without complaint. Options: `pgWorker` compares the stored
   controller UUID with the agent's at start and fails fatally on a mismatch
   (like D50), or only at bootstrap. Open choice; needs a decision before
   work.
4. Failure clean-up stays manual (D52); do not make bootstrap drop schemas.

Stop conditions: do not add a lock or a second ownership table before the
live check shows the existing guard is insufficient.

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

### Run 2 (2026-11-05, `09=14d1ea2f36`)

Prerequisites: U15 (juju/juju#23710, #23723) and U16 (juju/juju#23728)
cherry-picked onto Stage 5 under D43 on base `5259edff43`
([record](../UPSTREAM-FIXES.md#poc-reconciliation-onto-5259edff43-2026-11-05));
PostgreSQL server `lxd36-pgtest` provisioned with the Stage 1 script
(disposable profile); network boolean fix (05-BOOT-FIXES) applied first.

`make juju jujuagentd`, then `timeout 1500 juju bootstrap localhost pgpoc-pg
--config ~/.config/juju-pg-poc/bootstrap.yaml --build-agent` (0600 file with
`db-backend`, `db-url` and apt proxy settings; no `--debug`, which may log
the URL). Bootstrap returned 0, but `juju status` hung: machine agent login
failed with 42803 (`mci.instance_id` not grouped) in agentpassword
`MatchesMachinePasswordHashWithNonce`. U15 and U16 no longer appeared.

The four agentpassword tests that cover the query fail on PostgreSQL; the
package had never been run there. This prompted sweep 2
([Stage 6](06-compatibility.md#postgresql-sweep-2)).

Fixed on Stage 5 (`ada2a852ee`) and Stage 6 (`283a00f630`, now `fa75871904`). Teardown: `lxc
delete -f`, `juju unregister pgpoc-pg --no-prompt`, drop both schemas.

### Run 3 (2026-11-05, `09=ba634a0f13`)

Bootstrap returned 0 and `juju status` works: machine 0 started. The
`controller` application unit stays at "agent initialising": its uniter
fails with 42803 (`sre.revision_uuid`) in secret
`getSecretsRevisionExpiryChanges`. That is 05-SECRET-X16, deferred until
after the D42 gate; it was taken up then because it blocks bootstrap.

Fixed on Stage 5 by `5e70fcbc84`, `0ed13ff885` and `3a8f0090c2` (unitstate
`updateCharmSecret`, the next site on the uniter path). The run 3
controller's API still answered, but its unit stayed "agent
initialising". Teardown on 2026-11-05 was manual, as for run 2: `lxc
delete -f juju-ebeba2-0`, `juju unregister pgpoc-pg --no-prompt` and
`drop-juju-schemas.py --controller 4c1af278-94bd-4f9b-8e87-8a6608a37823`.
`destroy-controller` was not tried, because controller removal still has
open X16 sites (U22). The database was left with no Juju schemas.

### Run 4 (2026-11-11, `09=75223c9543`)

The final-tip 142-package sweeps completed with the known PostgreSQL
catalog-teardown failures and the dqlite nullable-BOOLEAN failure; see
[Stage 6](06-compatibility.md#final-tip-sweeps-2026-11-11). `make juju
jujuagentd` succeeded. The previous private bootstrap config pointed to an
old database host, so `~/.config/juju-pg-poc/bootstrap-75223c.json`
was generated mode 0600 from the current test database URL. Inventory
before bootstrap showed no controller, model or test schemas and no other
database sessions.

PostgreSQL bootstrap created controller and controller-model schemas and
started LXD instance `juju-c5b53b-0`, but the controller API never came up.
The bootstrap worker repeatedly timed out connecting to Charmhub while
resolving `juju-controller`; HTTPS probes from both host and container
timed out. The user confirmed Charmhub was down. This is an external
blocker, not evidence of a PoC code failure.

The bootstrap client was killed without running its default failure cleanup;
the instance and agent were left running. Controller UUID is
`9606321a-81ab-4f1f-8ad7-597769b998c9`; controller model UUID is
`74761cfd-a5bc-4dfb-8114-14e2f1c5b53b`. Both PostgreSQL schemas were
initialised, and the local controller entry `pgpoc-pg` existed. The
bootstrap client exited 137 and the run is **incomplete**; no status,
model workload or D54 scenario has been verified. Log:
`/tmp/pg-poc-75223c-bootstrap.log`.

Later, the user stopped and unregistered the controller. After the user
approved the specific targets, the controller-specific cleanup tool
dropped both schemas. A subsequent inventory showed no controller, model
or test schemas and no other sessions; `public` and `login_hook` remained.
Reassess a fresh bootstrap after Charmhub recovers.

### Bootstrap retry preparation (2026-11-11, `09=f3340c6710`)

The PostgreSQL and real-dqlite sweeps completed without failures at this
tip; see [Stage 6](06-compatibility.md#catalog-teardown-dependency-check-2026-11-11).
`make juju jujuagentd` succeeded, and the private bootstrap configuration
remained mode 0600. The database inventory showed no controller, model or
test schemas and no other sessions. Build log:
`/tmp/pg-poc-f334-bootstrap-build.log`.

Although `charmhub.io` responded over HTTPS, connections to
`api.charmhub.io` timed out from the host and an existing LXD controller.
All six IPv4 addresses returned by DNS timed out from the host. This
establishes an API connectivity problem from these locations, not whether
the service was globally unavailable. With user approval, the bootstrap
retry was deferred until API connectivity is restored; no controller or
bootstrap resources were created in this attempt.

### Run 4: prerequisites and next steps

Prerequisites, in order:

1. Done 2026-11-05: 05-unitstate-relation-key (`1afd3b9434`) and
   05-X16-rest (`52a0eea6ee`, `367f754dd6`, `640d3bcabf`), restacked to
   `09=760f3b675f` and re-run on PostgreSQL. `destroy-controller` may now
   be tried; removal controller has no X16 left.
2. Done at `09=a3818782aa`: the
   [PostgreSQL re-sweep](06-compatibility.md#postgresql-re-sweep) ran at
   `-p 8`. Triage its failures and skips for the functional workload.
3. Run 4 created fresh schemas and started the instance without prior
   cleanup, but could not complete during the Charmhub outage. The failed
   resources have since been cleaned up with user approval. After recovery,
   reassess bootstrap and the D54 scenario, then run the dqlite comparison
   from the same revision.

### Run 5: successful PostgreSQL bootstrap (2026-11-11, `09=f3340c6710`)

Charmhub's API returned HTTP 200 from the host and an existing LXD
controller. The configured PostgreSQL database had no controller, model
or test schemas and no other sessions. The PoC checkout was clean at
`f3340c67107829347ee750f7e40e806e563411e9`; `make juju jujuagentd`
succeeded with `GOMAXPROCS=2` and Go `-p 2`.

From the PoC checkout:

```bash
timeout --kill-after=30s 900s env GOMAXPROCS=2 GOFLAGS=-p=2 \
  /home/jameinel/go/bin/juju bootstrap localhost pgpoc-pg \
  --config ~/.config/juju-pg-poc/bootstrap-75223c.json \
  --config bootstrap-timeout=600 --build-agent --keep-broken --no-switch
```

The private config remained mode 0600; no database URL was put in command
arguments or recorded output. Bootstrap returned 0 and created LXD instance
`juju-c5f98c-0` at `10.158.110.124`. The built agent reports
`4.1-beta3.1`. `juju-controller` revision 482 on `4.1/stable` converged
to active, with `controller/0` active/idle and machine 0 started.

Read-only database inventory verified:

- `juju_controller`: initialised, 61 patches, controller UUID
  `f28bf008-6149-44c9-84a2-680c26b01c06`.
- `juju_7d1c9ee0-8da6-4f5a-81df-a92067c5f98c`: initialised,
  158 patches, model `admin/controller`, belonging to that controller.

The machine agent config explicitly records `dbbackend: postgresql`.
Restarting `jujuagentd-machine-0.service` succeeded; the API returned
status again, with the controller unit active/idle. The subsequent database
inventory retained the same controller/model UUIDs and patch counts.

Evidence:

- Build: `/tmp/pg-poc-f334-bootstrap-retry-build-20261111.log`.
- Bootstrap: `/tmp/pg-poc-f334-bootstrap-run5-20261111.log`.
- Initial status: `/tmp/pg-poc-f334-run5-controller-status.json`.
- Post-restart status:
  `/tmp/pg-poc-f334-run5-controller-status-after-restart.json`.
- Post-restart inventory:
  `/tmp/pg-poc-f334-run5-pg-inventory-after-restart.log`.

No workload models were created and no cleanup ran. `pgpoc-pg` remains
registered and running. This establishes bootstrap and controller-agent
restart, not the full functional milestone. Next: run D54's two-model
workload, cross-model secrets and approved teardown, then the same-revision
dqlite scenario. PostgreSQL tests and sweeps must not share this database
while the controller is using it.

### Run 5: D54 PostgreSQL workload (2026-11-11, `09=f3340c6710`)

The remaining evaluation started from the healthy run 5 controller, with
no code changes. Both models were created with `--no-switch`, and every
subsequent command explicitly selected `pgpoc-pg` or its model.

| Model | UUID | Applications / pinned artifacts |
|---|---|---|
| `model-secrets-offer` | `29a2db74-a30b-4b1e-8e2b-7be6c79fd9dd` | `ubuntu` revision 79, `latest/stable`, Ubuntu 24.04; `dummy-source` (`juju-qa-dummy-source`) revision 6, `latest/stable`, Ubuntu 22.04 |
| `model-secrets-consume` | `b2ae6c01-80e4-44da-811d-ef838db48cd5` | `dummy-sink` (`juju-qa-dummy-sink`) revision 7, `latest/stable`, Ubuntu 22.04, two units |

The offer was `admin/model-secrets-offer.dummy-source`, endpoint `sink`;
the consumer added it as SAAS `dummy-source` and related
`dummy-sink:source` to `dummy-source:sink`. Database inventories verified
both model schemas were initialised with 158 patches, alongside the
unchanged controller and controller-model schemas.

Passed checks, with each convergence wait bounded at 600 seconds:

1. All four workload units became idle. A real `juju exec` on `ubuntu/0`
   wrote `/tmp/d54-marker` and read back `d54-pgpoc-pg`.
2. Setting `dummy-source token=foobar` made the source active, propagated
   active status to the remote endpoint and left both sink units idle.
   The offer reported one active connection.
3. An application-owned secret with test content `foo=bar` was granted
   to the source relation. Both sink units read it by URI and label.
4. Rotating to `foo=bar2` triggered a `secret-changed` hook on each sink
   unit, measured against stable per-unit status-history baselines.
   Both units retained `bar` in their tracked revision, saw `bar2` with
   `--peek`, and retained `bar` after peeking.
5. Refreshing only `dummy-sink/0` left it tracking `bar2` while
   `dummy-sink/1` still tracked `bar`; the behind unit could peek `bar2`.
6. Restarting `jujuagentd-machine-0.service` restored an active/idle
   controller. The Ubuntu marker and each unit's different tracked
   revision survived; the behind unit could still peek and then refresh
   `bar2`. All four namespaces retained their UUIDs and patch counts.
7. Suspending the relation denied secret access to both sink units.
   Resuming restored access to both. Revoking the grant again denied both.
   The expected denied operations returned nonzero task codes with
   `permission denied`; they are successful negative checks, not unexpected
   failures.

The initial local evaluator incorrectly required stdout from every
successful exec task; `secret-grant` legitimately returned code 0 without
stdout. The evaluator was corrected to accept omitted stdout, the existing
secret was reused, and the secret evaluation passed. This was not a Juju
production failure; no code repository files were modified.

The reusable evaluator is a session artifact:
`/home/jameinel/.copilot/session-state/15a46cf0-e580-4d60-97ba-476508025f4a/files/d54.sh`.
Its `ready`, `secrets` and `recheck` phases were run for `pgpoc-pg`.
Evidence:

- Workload operation and token: `/tmp/pg-poc-d54-pg-ready-20261111.log`.
- Secrets: `/tmp/pg-poc-d54-pg-secrets-retry-20261111.log`.
- Restart, suspension, resume and revocation:
  `/tmp/pg-poc-d54-pg-restart-revoke-20261111.log`.
- Initial workload inventory:
  `/tmp/pg-poc-d54-pg-models-inventory-20261111.log`.
- Post-restart inventory:
  `/tmp/pg-poc-d54-pg-before-teardown-20261111.log`.
- Pre-teardown status:
  `/tmp/pg-poc-d54-pg-offer-before-teardown.json` and
  `/tmp/pg-poc-d54-pg-consume-before-teardown.json`.

**User-requested stop:** after reviewing the exact cleanup targets, the user
selected "Preserve these models and stop before teardown". No relation,
SAAS, offer, application, model or schema was removed. Both workload models,
their four LXD instances and both PostgreSQL schemas remain, along with
`pgpoc-pg`. The secret grant remains revoked.

At this stop, D54 was **partially verified**, not complete: normal
relation/application/model teardown and schema isolation remained untested,
and `pgpoc-dqlite` had not been bootstrapped. The user subsequently authorised
the dqlite comparison without PostgreSQL teardown and approved mixed CMR in
both directions; that continuation is recorded below. Cleanup still needs
explicit target approval. Do not run PostgreSQL test sweeps against this live
database.

### Dqlite comparison and bidirectional mixed CMR (2026-11-11, `09=f3340c6710`)

The user authorised continuing without PostgreSQL teardown, and explicitly
selected both mixed-backend directions. All tests used the unchanged source
revision `f3340c67107829347ee750f7e40e806e563411e9`, the built client and
real agent binaries. No code repository files were modified.

| Scenario | Observed result |
|---|---|
| PostgreSQL, same controller | Previously recorded workload, both-unit secrets and agent restart checks passed; preserved models still active/idle |
| Dqlite, same controller | Exec, secret and restart checks passed; final token-status check failed for `dummy-sink/1`, so the full workload is not verified healthy |
| PostgreSQL offerer -> dqlite consumer | Token propagation, both-unit secrets, separate restarts of both controllers and access-control checks passed |
| Dqlite offerer -> PostgreSQL consumer | Token propagation, both-unit secrets, separate restarts of both controllers and access-control checks passed |

#### Dqlite controller and original D54 workload

From the PoC checkout:

```bash
timeout --kill-after=30s 900s env GOMAXPROCS=2 GOFLAGS=-p=2 \
  /home/jameinel/go/bin/juju bootstrap localhost pgpoc-dqlite \
  --config bootstrap-timeout=600 --build-agent --keep-broken --no-switch
```

Bootstrap returned 0. Controller UUID:
`7b2b3422-2fd3-433d-8f64-21c9fce9c97b`; instance `juju-1c1412-0`,
address `10.158.110.187`; controller-model UUID:
`20b068f6-ef10-4ec3-8a1b-09a0a11c1412`. The agent reports
`4.1-beta3.1`, and the controller charm is revision 482 on `4.1/stable`,
matching PostgreSQL. Its machine config explicitly says `dbbackend: dqlite`;
`/var/lib/juju/dqlite` contains Raft segments and metadata. No PostgreSQL
bootstrap config was supplied.

The D54 workload used the same charm revisions, channels, bases and unit
counts as PostgreSQL. Its two model UUIDs are:

- `model-secrets-offer`: `10a2ba32-091e-4cec-8712-bcf61717004a`.
- `model-secrets-consume`: `fc34b97f-64f9-416f-8888-27b436a94f2c`.

The evaluator's `deploy`, `ready`, `secrets` and `recheck` phases completed.
The Ubuntu exec/filesystem check, secret reads by URI/label on both sink
units, per-unit rotation hooks, cached/peek/refresh semantics, mixed tracked
revisions across controller-agent restart, suspension/resume and revocation
all passed. These were correctness checks, not performance measurements.

**Unresolved final-state finding:** the stricter final status assertion
required every workload unit to be active/idle. In
`pgpoc-dqlite:model-secrets-consume`, `dummy-sink/0` reports active/idle
with "Token is foobar", but `dummy-sink/1` reports waiting/idle with
"Waiting for token". Its status has a `since` time of 19:38:10-05:00 and
was still present in the final 20:32 snapshot. The scripted readiness check
had required the remote endpoint to be active and both sink agents idle;
that did not establish token convergence on both units. The successful
secret checks do not waive this workload failure.

No root cause was established; do not attribute this to dqlite, PostgreSQL,
the charm or a particular SQL query without further evidence. After the
finding was reported, the user selected "Record the finding and preserve all
resources". No corrective config changes, forced hooks, code fixes or
teardown were attempted. The IDE had no debug session/configuration targeting
the affected running LXD agent; no debugger was started.

#### Isolated mixed-backend relations

Each direction used separate models, one pinned `juju-qa-dummy-source`
revision 6 and two `juju-qa-dummy-sink` revision 7 units, all on Ubuntu 22.04
and `latest/stable`. Existing D54 models/relations were not reused or removed.

| Controller | Model | UUID |
|---|---|---|
| `pgpoc-pg` | `model-secrets-offer-pg-to-dqlite` | `f7e06a7e-79b8-44b2-8d69-63a74c2b29bc` |
| `pgpoc-dqlite` | `model-secrets-consume-pg-to-dqlite` | `1a87b74c-3d7f-448d-8dfb-5dabd5213979` |
| `pgpoc-dqlite` | `model-secrets-offer-dqlite-to-pg` | `f49fda52-9ad3-4f63-87cd-b0daa27e1af7` |
| `pgpoc-pg` | `model-secrets-consume-dqlite-to-pg` | `e662590c-fae4-4ac8-80ed-36dca86b4b64` |

Each source offered `dummy-source:sink`. The opposite controller consumed
the fully qualified offer as SAAS `dummy-source` and related
`dummy-sink:source` to `dummy-source:sink`. Setting `token=foobar` propagated
to both sink units. Final snapshots confirmed active/idle and "Token is
foobar" for every mixed-model source and sink unit.

Both directions passed the same two-unit secret read, rotation hook,
cached/peek and mixed-revision checks as D54. With unit 0 refreshed and unit 1
still behind, the PostgreSQL controller agent was restarted and both
directions rechecked; then the dqlite controller agent was restarted and both
directions rechecked again. Each unit's tracked revision survived each
restart, and the behind unit could still peek the newest revision.

Finally, unit 1 refreshed; suspending each offering relation denied both
consumer units, resuming restored access, and revoking each grant denied
both again. All checks returned their expected results. Negative access
checks deliberately produced nonzero task codes with `permission denied`.

The PostgreSQL inventory retained the original four namespaces and showed
the two added mixed-model schemas initialised with 158 patches each.
Controller/model UUIDs were unchanged after the restarts.

#### Evidence and retained state

The session evaluator now accepts the offering controller, phase, optional
consuming controller and model-name suffix. Mixed runs used:

```bash
bash "$EVAL" pgpoc-pg deploy-cmr pgpoc-dqlite -pg-to-dqlite
bash "$EVAL" pgpoc-dqlite deploy-cmr pgpoc-pg -dqlite-to-pg
```

Here `EVAL` is the session artifact path recorded above. The corresponding
`cmr-ready`, `secrets`, `cmr-persist` and `cmr-recheck` phases produced:

- Dqlite comparison:
  `/tmp/pg-poc-d54-dqlite-{bootstrap,deploy,ready,secrets,restart-revoke}-20261111.log`.
- PostgreSQL -> dqlite:
  `/tmp/pg-poc-cmr-pg-to-dqlite-{deploy,ready,secrets,pg-restart,dqlite-restart,revoke}-20261111.log`.
- Dqlite -> PostgreSQL:
  `/tmp/pg-poc-cmr-dqlite-to-pg-{deploy,ready,secrets,pg-restart,dqlite-restart,revoke}-20261111.log`.
- Model inventories:
  `/tmp/pg-poc-cmr-{pgpoc-pg,pgpoc-dqlite}-models-20261111.json`.
- PostgreSQL namespace inventory:
  `/tmp/pg-poc-cmr-mixed-pg-inventory-20261111.log`.
- Final workload statuses:
  `/tmp/pg-poc-final-<controller>-<model>-20261111.json`, including the
  unresolved original dqlite consumer in
  `/tmp/pg-poc-final-pgpoc-dqlite-model-secrets-consume-20261111.json`.

All eight workload models, their applications, relations, offers and SAAS,
both controllers and all six PostgreSQL namespaces remain. Test secret
grants are revoked; relations were resumed. No teardown ran on either
backend. Full D54 acceptance still requires investigating the original
dqlite token-status finding and separately authorised teardown/schema
isolation checks. Both mixed-backend CMR directions passed the checks above;
this is not evidence for HA, model migration or performance claims.

#### Follow-up: original dqlite sink token investigation

Observed through 2026-11-12 02:49 UTC, continuing the local 2026-11-11
evaluation. No agents were restarted, hooks retriggered, relations changed
or resources removed during this investigation.

The initial dqlite readiness snapshot and unit logs show both sink units
active with `Token is foobar` at 00:36:55 UTC. During suspension they
processed relation-departed/broken at 00:38:09. On resume, both processed
relation-created/joined and their first relation-changed at 00:38:10.
Both first changed hooks read an empty token and set waiting status.
Unit 0 then processed another changed hook, read `foobar` and became active.
Unit 1 did not process that recovery hook.

The installed revision-7 hook runs `relation-get token`, writes the result
to `/var/run/dummy-sink/token` and sets its status from that result. A later
read-only `juju exec` comparison found an empty last-hook token file on
unit 1, but an explicit `relation-get -r source:0 - dummy-source/0` returned
`token=foobar` on both units. This is not merely an incorrect status message.

At the user's request, direct database reads compared the original
`model-secrets-consume` models, without attaching a debugger. PostgreSQL was
queried inside a read-only transaction using the existing credential-safe
psql helper. Dqlite was queried through the agent's read-only database REPL,
opening model `fc34b97f-64f9-416f-8888-27b436a94f2c`; no live SQLite files
were opened directly.

Both databases contain an alive, unsuspended relation 0, relation-unit rows
for both sinks and the source, and `token=foobar` in the source unit's
`relation_unit_setting` rows. The sinks' own settings contain their network
addresses, not the token; the token belongs to the remote source databag
and is shared by both consumers.

| Backend | Source settings-hash prefix | Current settings version | Sink 0 recorded version | Sink 1 recorded version |
|---|---|---|---|---|
| dqlite | `e3d84ef9` | `4136104582745174885` | same | same |
| PostgreSQL | `d48149de` | `7306027391828636772` | same | same |

Full source settings hashes:

- dqlite: `e3d84ef9aeb09a45a22e7cb16154a3176fcf83c8dcda8d347a638186dffb3e7c`
- PostgreSQL:
  `d48149de593a0240be30e5405efbdabe2e7bd9c61444e37e67e2150b9b5452d9`

The versions above use `domain/relation/state/relation.go`'s `hashToInt`:
the first eight bytes of the hash string interpreted little-endian. Both
`unit_state_relation` records already acknowledge the current source
settings version, including the waiting dqlite unit. Neither has a
changed-pending entry. Both uniter states are at `op: continue`,
`opstep: pending`, rather than a pending or failed hook.

The database therefore says unit 1 should have access to the token and
already records its current version as delivered. The relation resolver
compares remote and recorded versions, so equal versions do not generate
another changed hook. This narrows the unresolved failure to consistency
between hook delivery, its recorded version and the settings it read;
the token is not missing from the current database. These snapshots do not
establish which watcher/context transition caused that inconsistency, or
prove that it is backend-specific.

Final status still showed dqlite unit 0 active/idle, dqlite unit 1
waiting/idle, and both PostgreSQL sinks active/idle. Both controller agent
services remained active with their earlier start times.

Additional evidence:

- `/tmp/pg-poc-sink-investigation-unit{0,1}-log-20261111.log`
- `/tmp/pg-poc-sink-investigation-relation-get-{0,1}-20261111.json`
- `/tmp/pg-poc-sink-investigation-pg-db-20261111.log`
- `/tmp/pg-poc-sink-investigation-dqlite-db-20261111.log`
- `/tmp/pg-poc-sink-investigation-db-version-comparison-20261111.log`

The local log paths are supporting artifacts, not files included in this
repository. The timeline, selected database values and unresolved conclusion
above are retained here so the finding does not depend on those artifacts
remaining available.

#### Read-only database comparison queries

For PostgreSQL, use the existing credential-safe psql helper with a
`BEGIN READ ONLY` transaction, bounded statement timeout and
`SET LOCAL search_path TO "juju_b2ae6c01-80e4-44da-811d-ef838db48cd5"`.
Do not print the database URL or run test suites against this live database.
For dqlite, use `jujuagentd db-repl --machine-id 0` on `juju-1c1412-0`
and `.open fc34b97f-64f9-416f-8888-27b436a94f2c`. Query through the
read-only cluster connection, not a copy of the live SQLite files.

These queries work in either selected model database:

```sql
SELECT
    r.relation_id, r.suspended, u.name, s.key, s.value, h.sha256
FROM relation AS r
JOIN relation_endpoint AS re ON re.relation_uuid = r.uuid
JOIN relation_unit AS ru ON ru.relation_endpoint_uuid = re.uuid
JOIN unit AS u ON u.uuid = ru.unit_uuid
JOIN relation_unit_setting AS s ON s.relation_unit_uuid = ru.uuid
LEFT JOIN relation_unit_settings_hash AS h
    ON h.relation_unit_uuid = ru.uuid
WHERE r.relation_id = 0
    AND u.name = 'dummy-source/0'
    AND s.key = 'token';

SELECT
    u.name, s.key, s.value
FROM unit_state_relation AS s
JOIN unit AS u ON u.uuid = s.unit_uuid
WHERE u.name IN ('dummy-sink/0', 'dummy-sink/1')
    AND s.key = '0'
ORDER BY u.name;
```

The first query must return `foobar` and the corresponding source hash.
The second returns each sink's YAML state; compare `members.dummy-source/0`
with the hash-derived version above. Equal versions alone are not workload
acceptance: check each unit's actual token status independently.
