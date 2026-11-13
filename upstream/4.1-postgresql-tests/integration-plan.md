# PostgreSQL schema validation into upstream 4.1

Approved direction, 2026-11-12. Owner/session:
`0ec926e6-3858-4f43-bc9c-658024127e88`.

[Branch status](status.md) | [Dashboard](../../UPSTREAM-PLAN.md) |
[SQL-regression contract](../../PLAN.md)

## Outcome and boundaries

PostgreSQL acceptance is a schema and SQL correctness check, not an optional
testing curiosity. SQLite accepting a query does not establish that its types,
keys, grouping or ordering are correct. Legitimate dialect differences still
need explicit treatment.

The deliverable is an upstream-4.1 SQL-regression harness, with checked-in
PostgreSQL schemas and a reproducible GitHub job. It is not PostgreSQL controller
runtime support: no controller configuration, bootstrap charm, runtime database
selection, LISTEN/NOTIFY worker, or PostgreSQL operator.

Start from stock upstream, not PoC ancestry. Reference PoC implementations are
inputs to review, not commits to import wholesale. Implementation base recorded
on 2026-11-12:
`b36eefc4b34badcdef972c859e96ccb974c592e2`. The changes since the previous
`bd970f693c` base do not touch the relocated schema or harness configuration
files; they do include other code and a `go.mod` change, so validation must be
rerun on the refreshed base.

## Branch graph and ownership

```text
upstream/4.1
  |
  +-- 4.1-schema-layout                 independent prerequisite PR
        |
        +-- 4.1-postgresql-tests        schema generator + shared SQL harness
              |
              +-- 4.1-postgresql-tests-ci    PostgreSQL job and CI wiring
                    |
                    +-- 4.1-postgresql-tests-integration
                          + exact pending SQL/schema fix heads
                          + regenerated PostgreSQL output
```

The integration line is disposable, not a fourth mergeable feature. The layout
branch is a short-lived prerequisite, not another permanent testing workstream.
Pending correctness fixes remain independently reviewable branches based on
stock 4.1; the diagram does not make them depend on infrastructure.

Create local CI/integration refs after the initial harness checkpoint to reserve
the agreed workstreams. Initially those refs contain no CI or combined-fix
implementation. Their existence is not completion evidence. Only advance refs
owned by this session; do not rebase or change another worker's branches.

Local checkpoint created on 2026-11-12:

| Branch | Head | State |
|---|---|---|
| `4.1-schema-layout` | `9da8100e161f59ce5a86485e0725db1b8101154d` | Move-only commit, clean worktree. |
| `4.1-postgresql-tests` | `6e677c28505645413b55862e732c28e576ced3b9` | H1 committed on layout, clean worktree. |
| `4.1-postgresql-tests-ci` | `6e677c28505645413b55862e732c28e576ced3b9` | Reserved ref only; no CI implementation. |
| `4.1-postgresql-tests-integration` | `6e677c28505645413b55862e732c28e576ced3b9` | Reserved ref only; no combined fixes. |

The user subsequently approved pushing these four exact heads on 2026-11-12.
All four were pushed atomically to `jameinel/juju` and verified against GitHub.
No PR was created. Planning documents were left uncommitted in the shared
planning checkout at that time and were not part of that push.

Keep plans, source manifests, QA evidence and local PR drafts in this planning
repository. Do not copy planning records into the Juju source repository.
Publication requires the user's review of the exact destination, content and
action, followed by explicit approval.

## 1. Independent schema-layout prerequisite

Branch: [`4.1-schema-layout`](../4.1-schema-layout/status.md).

One mechanical commit moves these complete trees:

```text
domain/schema/controller/ -> domain/schema/dqlite/controller/
domain/schema/model/      -> domain/schema/dqlite/model/
```

Update imports, embeds, patch discovery, existing trigger-generation
destinations, `ddlgen` release-output paths, workflow paths and documentation.
Preserve `schema.ControllerDDL()` and `schema.ModelDDL()`, patch ordering and
contents, hash chains, trigger contents and historical release DDL bytes.
Historical generated comments are not rewritten just to modernize paths.

Do not add SQL conversion code, PostgreSQL DDL, environment selection or schema
fixes here. PoC commit `11c1c182dd` supplies only the move patch, applied to the
refreshed stock base. Verify all 100 relocated files against upstream Git blobs
before and after generation, inspect every remaining old-path reference, and
exercise existing fresh/idempotent/cumulative-schema behavior and direct
consumers on SQLite and real dqlite. Format/lint the changed Go wiring; stress
the database lifecycle exercised by the shared harness.

This PR can land independently, before the PostgreSQL design is finished.

## 2. Infrastructure branch: explicit implementation checkpoints

Branch: [`4.1-postgresql-tests`](status.md). Stack on layout temporarily; after
layout lands, rebase the infrastructure-only commits onto refreshed upstream.

### H1. Central backend selection

Implemented in local commit `6e677c2850`, on the refreshed layout base:

- `JUJU_TEST_DATABASE` selects the backend. Unset preserves the compiled
  SQLite/dqlite default; unavailable or unknown explicit backends fail.
- `JUJU_TEST_POSTGRESQL_URL` can remain configured while switching backends.
  Ignore it unless PostgreSQL is selected; do not log its contents.
- Read configuration centrally, preserve it through environment isolation and
  consume it in the shared suite. New domain suites need no environment logic.

The current PostgreSQL path deliberately reports "not implemented yet". This
checkpoint is not a landable, complete PostgreSQL implementation.

### H2. Driver, error classification and schema application

Port the minimum PostgreSQL test driver and schema-application plumbing. Reuse
existing transaction/SQL abstractions, and classify PostgreSQL errors alongside
existing SQLite/dqlite errors. Exercise constraint failures, retries,
cancellation, commit/rollback, NULL/BOOLEAN values, parameters and cleanup.
Do not hide domain SQL failures in the driver through query rewriting.

Keep production database selection unchanged. Tests must identify the backend
actually opened, not just echo a configuration value. Backend-aware schema
application must use the correct assets, transaction boundaries and bookkeeping.
Record and verify the applied schema version/hash rather than merely returning
success after executing statements.

Define the PostgreSQL version/upgrade contract before completing this step.
The first harness may target fresh regression databases, but it must account
for every registered patch needed by the current schema. A translator that
rejects `.PATCH.sql` cannot silently omit registered post-patches or claim
PostgreSQL upgrade coverage. Dqlite's supported upgrade behavior stays intact.

### H3. Generation and first-class checked-in schemas

The authoritative existing SQL and trigger definitions stay under dqlite and
the schema assembly package. Add `generate/pgschemagen` and checked-in output:

```text
domain/schema/dqlite/{controller,model}/
domain/schema/postgresql/{controller,model}/
```

No translation during suite setup. A deterministic generation/freshness check
must detect stale output and stale overrides. Translate the assembled schema,
including Go-built triggers, in a dependency-valid order. Fail explicitly on
unsupported SQL and unresolved references. Test genuine dialect changes:
types, identifiers, identity/sequence behavior, triggers, deferred references
and timestamps. Preserve change-log visibility/ordering guarantees; a sequence
plus advisory-lock trigger is substantive behavior requiring concurrency tests.

Generate against the actual refreshed 4.1 source. It still has seven
`STRFTIME` defaults, whereas the latest PoC generator removed that translation
after a larger schema redesign. Handle those timestamp semantics deliberately;
do not import PoC D56's charm insertion-order/default redesign incidentally.

The PoC has seven manual override files, totaling eleven hunks. Audit each:

| Concern | Treatment |
|---|---|
| Predicate-free joins in model, secret, unit and CMR views | Shared portability fix candidates; do not quietly bless them as permanent PostgreSQL exceptions. |
| Last-login bare `time` beside an aggregate | Shared grouping/semantic fix, with a precise expected result. |
| Nullable constraint BOOLEAN | Coordinate the shared schema/type correction with the nullable-BOOLEAN driver fix. |
| Constraint tag/space/zone `rowid` emulation | Establish explicit shared ordering; a PostgreSQL-only identity is not a substitute for that contract. |
| Duplicate foreign-key constraint name | Review as a shared schema fix where appropriate. |

Also audit automated `IIF`, `IFNULL`, Boolean literal and reserved-identifier
rewrites. Distinguish dialect translation from masking a shared defect. Keep
any genuinely unavoidable compatibility override narrow, documented, exactly
matched and linked to its removal condition. Do not make infrastructure green
by accumulating silent translations of incorrect shared SQL.

### H4. Shared suite lifecycle and end-to-end coverage

Replace H1's explicit PostgreSQL rejection with real driver/schema setup in
the shared database suite. Use bounded connection pools, unique run-owned
namespaces/databases and deterministic cleanup on setup failure and teardown.
Never sweep unrelated databases or rely on controller PID identity to delete
objects on an arbitrary shared PostgreSQL server. Test concurrent suites,
cancellation, partial setup and pool/namespace cleanup under race and stress.

Keep existing suite constructors/API behavior compatible. Carry the selected
backend through model/controller setup, schema application, transactions and
consumers; an unchanged domain suite must exercise PostgreSQL without bespoke
configuration. No silent fallback, added skips or weakened assertions.

The previously deferred changestream shutdown cancellation fix
(`7e513ceaa8`) belongs with or after this PostgreSQL suite work. Port that
bounded fix and its direct/wrapped-cancellation regression deliberately; do not
let a PostgreSQL shutdown failure disappear into retries or blanket log ignores.

## 3. GitHub PostgreSQL-job branch

Branch: `4.1-postgresql-tests-ci`, based on infrastructure, not on pending fixes.

Provision a disposable PostgreSQL service, wait for readiness, supply the two
approved variables centrally, run the SQL-regression entry point, retain
diagnostics without credentials, and clean up on success, failure and timeout.
Pin the service image/version and document resource settings and runner hardware
when chosen; those details are not decided by this plan. Keep existing
SQLite/dqlite jobs and required checks intact.

The job must include provisioning, compilation, execution and cleanup in its
timing. Acceptance is under 30 minutes, preferably under 15, with hardware and
cold/warm-cache conditions recorded. Warm-cache PoC package timings alone are
not acceptance evidence. Bound job concurrency and fail early with useful
diagnostics rather than hide failures behind retry loops.

Do not make the PostgreSQL check required while its result still depends on
unmerged fixes or has unclassified failures. Prepare workflow text locally;
publishing or changing repository branch protection needs separate approval.

## 4. Disposable integration branch and source manifest

Branch: `4.1-postgresql-tests-integration`. Its initial ref is only a reservation.
Build the real demonstration from the current CI head plus exact approved
breakout heads, preserving provenance. Never use the entire PoC as its base.

The following is the initial remote-head inventory read from `juju/juju` open
4.1 PRs on 2026-11-12, not a claim that those commits are integrated or approved.
Refresh before construction; local breakout refs may be behind their PR heads.
Respect branch-owned holds and dependency order. In particular, do not treat
the storage-attachment semantic hold as resolved by this plan.

| Breakout | Upstream PR | Inspected remote head |
|---|---|---|
| `4.1-model-config-precedence` | juju/juju#23683 | `f50dfe4257d1627711ab6f65ce013b4504f3af5c` |
| `4.1-crossmodel-secret-revisions` | juju/juju#23688 | `aa6cad849cb522088ac444a32eb939cf7efdea85` |
| `4.1-operation-query-portability` | juju/juju#23693 | `0fb2eb7194e484c20d1b5d3d2a9dcbf0fab5664b` |
| `4.1-nullable-primary-keys` | juju/juju#23710 | `838abf41ea3645d211f566de6d85cf8c640a3908` |
| `4.1-application-exposure-scope` | juju/juju#23723 | `4e66f7c4497fdf1a9843fb2f236ee4f25f639b5d` |
| `4.1-lookup-key-not-null` | juju/juju#23724 | `4e7341392fdcb0490f30b86d76e809f69ed1bdaf` |
| `4.1-lease-pin-conflict` | juju/juju#23729 | `c55d5839e5516157d5017b66ece7655861f2b61e` |
| `4.1-sequence-upsert-target` | juju/juju#23731 | `696c116f7943d16e6d1706fc857c561f90ee8aa3` |
| `4.1-cte-membership-subqueries` | juju/juju#23750 | `67daa565a60f9ab9d2f9b5ea43a3772ce8528fa3` |
| `4.1-drop-iif` | juju/juju#23751 | `fcec930751a357aa60f4c415e63d1bad9f446760` |
| `4.1-dqlite-nullable-boolean` | juju/juju#23759 | `39b1dc3cd8afcbb9eb60d44eab62101974a66dc9` |
| `4.1-controller-export-null` | juju/juju#23765 | `3b733c237bc98f8669307dae42091469c579542d` |
| `4.1-storage-filesystem-attachment-row` (held) | juju/juju#23766 | `2bfbdc7d6077364266c0756ad9898dd8a830d8ee` |
| `4.1-unitstate-relation-key` | juju/juju#23769 | `9a3083a887952914d1457d10084bacfa78bd0c6d` |
| `4.1-x16-grouped-columns` | juju/juju#23784 | `c3161fb4ff81a08626ce2a6a1ef99f4645ee5c2a` |
| `4.1-cmr-ingress-order` | juju/juju#23805 | `1f2411ad8c013246780abcc0e1077db179d51197` |
| `4.1-controllerupgrader-stream-result` | juju/juju#23806 | `a380152f9beabb42bd64cfeb2dfff252439a6b9f` |
| `4.1-ssh-host-key-conflict` | juju/juju#23807 | `e00ed96510b49abd0a1d78efc1d99e20cf6495f7` |
| `4.1-boolean-and-aggregate-literals` | juju/juju#23809 | `5f8af01651bd23706c22b9e1aa326b94206cb423` |

PR-description guidance and dependency-installation work are separate concerns,
not SQL correctness patches to import indiscriminately. Schema candidates still
listed as "not started" in the dashboard are not covered by this PR inventory.
Allocate them as upstream fixes when failures establish the need; do not fill
that gap with PoC production changes on the infrastructure branch.

Record each actual integration input's base, exact head, owner status and
dependency/hold. Prefer merges preserving input provenance; do not replay a
stack's prerequisite twice. Resolve renamed paths explicitly, and regenerate
PostgreSQL SQL from the combined dqlite source rather than hand-merge generated
output. Review the resulting shared-schema diff and remove obsolete overrides.

Classify every failure as baseline/shared SQL defect, infrastructure defect,
translation defect, or genuinely backend-specific test contract. Reproduce
ambiguous failures with the same package/test on both backends. Send shared
fixes back to their own branches. Record blockers; do not skip them to obtain a
green integration result.

## 5. Acceptance gates and landing sequence

Coverage is the full N16 SQL-owning inventory, not merely `./domain/...` or a
selection of convenient state packages. Commit a reproducible package-selection
mechanism with the infrastructure/CI work, and audit it against:

- Domain state/bootstrap packages and watcher-bearing domain root packages.
- Schema generation/application and core/internal database infrastructure,
  changestream and eventsource SQL consumers.
- Agent bootstrap and dbaccessor, upgradedatabase and leaseexpiry workers.
- SQL-owning import/export and other test suites found by the inventory audit.

For truly dqlite-specific replication/recovery/protocol tests, document an exact
backend-specific exclusion and rationale rather than adding broad skips.
Exercise both model and controller databases, schema failure paths,
transaction/error classification, stable ordering, watcher wakeups, cleanup and
parallel isolation. Run with race detection and bounded stress where required.
Existing SQLite/dqlite behavior, generation freshness, static analysis and
transaction checks must remain intact.

Landing order:

1. Land the independent layout PR after its own review and verification.
2. Continue landing shared correctness fixes independently. Refresh the
   integration manifest as each merges; resolve held semantics separately.
3. Finish H2-H4 and the CI branch. The disposable combined run establishes
   readiness only for the exact recorded inputs, coverage and job budget.
4. Once required fixes are in upstream 4.1, rebase infrastructure onto that
   stock head, regenerate from it, and repeat the same complete matrix. The
   infrastructure review diff must contain no unmerged shared fixes.
5. Land infrastructure, then rebase and land CI after its actual job passes.
   Enable a required check only when its result is trustworthy.
6. Retire/rebuild the demonstration branch as upstream advances. Never merge
   the mixed integration branch wholesale.

At each publication boundary, prepare a PR description from the root template,
with prospective merge-message rationale and reproducible functional QA steps.
Show the exact local draft and destination for approval before publishing.

Stop and obtain a decision for a new semantic/schema redesign, unresolved
release-history/upgrade contract, required unsupported DDL, held breakout,
cross-owner conflict, or a proposed scope/coverage reduction. A partial green
package run or a reserved branch ref does not satisfy these gates.
