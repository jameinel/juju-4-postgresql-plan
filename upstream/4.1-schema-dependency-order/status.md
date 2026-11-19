# 4.1-schema-dependency-order

[Shared direction](../../AGENTS.md) |
[Upstream dashboard](../../UPSTREAM-PLAN.md) |
[PostgreSQL test infrastructure](../4.1-postgresql-tests/status.md)

## Current state

Owner/session: `298b7b55-c2d5-4ee7-ba8c-573f1ced04fc`, authorized
takeover and exclusive reservation window confirmed by the user on
2026-11-18. Branch/worktree: `4.1-schema-dependency-order`,
`../4.1-schema-dependency-order`. Base:
`d380e5e4fba4c9918a0774638ffa3138aeece73e` (upstream 4.1 checked
2026-11-18); code head:
`c8282cfab478973d6d92727b2b19c3980f68f84b`. Implemented and locally
validated. Pushed to `jameinel/juju:4.1-schema-dependency-order` with
user approval and the remote head verified on 2026-11-18. Published
[juju/juju#23890](https://github.com/juju/juju/pull/23890) with user
approval; its base, head and PR body were verified on 2026-11-18.

The user accepted the live schema-comparison QA as passed on 2026-11-18:
the controller and model definitions matched apart from the documented
comment whitespace. Workload deployment and normal teardown were not
established by that run.

Next: monitor PR review and CI. The live QA provisioning failure below
remains unexplained, but is not a blocker to the user's acceptance of the
schema comparison.
Reconcile source moves with the schema-layout owner and independently
reviewed changes when they meet; the PostgreSQL harness must consume the
result separately. Do not treat this branch as their integration.

Release-policy check, 2026-11-18: upstream `4.1` identifies as
`4.1-beta3`; GitHub lists `v4.1-beta1` as its only 4.1 tag and both
post-patch registries are empty. Merged juju/juju#23682 explicitly
records that earlier 4.1 beta controllers are unsupported for in-place
upgrade across base-schema hash changes; juju/juju#23690 records the
same limitation for model databases. Reordering also changes the hash
chain, so it must not be presented as an in-place beta upgrade. Preserve
historical release DDL and validate supported migration paths separately.

Prepared 2026-11-18 by session
`5eb9907f-1835-4e64-98a2-d54e0aa2461d`, with the user's authorization
to split source ordering into an independent UPSTREAM task. The user
authorized handoff to this session on 2026-11-18.

At preparation, verified upstream 4.1 base:
`d380e5e4fba4c9918a0774638ffa3138aeece73e`, checked through GitHub on
2026-11-18. Created branch/worktree: `4.1-schema-dependency-order`,
`../4.1-schema-dependency-order`, with `_deps -> ../../_deps` sharing
the existing dependencies. The initially clean worktree then had no cleanup
commit, push or PR.

The existing dirty `4.1-postgresql-tests` worktree is unchanged. Its
same-patch foreign-key relocation checkpoint is not part of this task.
The user chose source cleanup followed by a strict generator, rather than
expanding automatic PostgreSQL dependency relocation.

An open-PR title/search check for schema work against 4.1 found no matching
ordering cleanup. Existing schema correctness PRs remain separately owned;
that bounded check is not an exhaustive audit of every open PR.

## Problem and evidence

SQLite permits tables and views to refer to objects defined later.
The current source order relies on that behavior. Removing forward
references at the source establishes an explicit construction contract on
the supported backends without adding PostgreSQL SQL or runtime machinery.

The audit applied every assembled controller/model patch to fresh SQLite
databases, queried actual foreign keys through `pragma_foreign_key_list`,
and prepared every view with a SQLite authorizer recording table/view
reads. Object creation positions came from `sqlite_master`. This audit
does not validate foreign-key target key/index timing, data-write ordering,
trigger dependencies or release compatibility; implementation must cover
those separately.

| Observed dependency | Controller | Model |
|---|---:|---:|
| Distinct forward table-to-target FK edges | 4 | 34 |
| Of those, cross-file edges | 2 | 27 |
| Multi-object cycles in the table/FK graph | 0 | 0 |
| Multi-object cycles including view-read dependencies | 0 | 0 |
| Self-referencing tables | 1 (`user`) | 0 |

The valid `user.created_by_uuid -> user.uuid` self-reference can remain
inside its `CREATE TABLE`; it does not need relocation.

Whole-file renaming alone cannot fix the model schema. FK dependencies
form a patch-level component spanning blockdevice, storage, constraint,
machine, application, unit and relation files even though the underlying
table graph has no multi-table cycle. Examples, using stock-4.1 paths:

- `model/sql/0019-application.sql:104-136` defines exposure tables that
  reference `application_endpoint`; `model/sql/0024-relation.sql:6-13`
  defines that endpoint table referencing `application`.
- `model/sql/0014-constraint.sql:22-24` references `container_type`,
  defined in `model/sql/0018-machine.sql:264`; the machine file also
  defines `machine_constraint`, which references `"constraint"`.

Views add more ordering requirements. Examples include `v_model_state`
before `model_migration_import`, `v_space_subnet` before subnet tables,
`v_application_charm_download_info` before `application`, and agent-version
views before `agent_version`. Preserve each view's SQL semantics; this task
does not fix predicate-free joins, aggregation, `IIF` or `rowid` usage.

Evidence was collected against harness head
`c59a24d92ab167313f4393e0651f5ca76c1efcc4`, whose authoritative SQL matches
the stock base apart from the layout relocation. Both controller/model SQL
directories were compared byte-for-byte with the new stock-base worktree.
Paths in the retained log therefore include `dqlite/`; this task starts
from stock paths.
Session artifacts: `files/schema-dependencies.go` and
`files/schema-dependencies.log` in the preparing session's directory.
The audit executed on SQLite only, not real dqlite or PostgreSQL.

## Approved scope and boundaries

- Start independently from stock upstream 4.1, not the PostgreSQL harness,
  PoC, schema-layout branch or pending correctness fixes.
- Reorder statements and rename/split SQL files under
  `domain/schema/{controller,model}/sql/` where necessary. Keep related
  comments, indexes and seed writes with their definitions where practical.
  Prefer the smallest understandable source organization over a runtime
  dependency solver or a broad schema redesign.
- Require referenced tables and matching primary/unique keys to exist
  before an FK is declared, allowing valid self-references. Require view
  sources to exist before the view is created, and trigger dependencies to
  exist before trigger creation.
- Add focused schema tests that enforce those invariants in assembled
  statement order, including within a file. Checking only end-of-patch or
  final-schema validity would miss the reported problem.
- Preserve table/column/constraint/index/view/trigger definitions and seed
  data. Do not remove constraints, defer their enforcement, rewrite joins
  or expressions, change stored types, introduce casts, or modify domain
  query behavior. Preserve relevant ordering of writes and side effects.
- Keep existing schema APIs, version-selected post-patch registration and
  historical release-DDL behavior. Do not add a second patch registry.
- Regenerate directly affected artifacts only when required by existing
  tooling; prove the resulting schema and seed data are unchanged.
- No PostgreSQL driver, selector, generated assets, CI, Sqlair dependency,
  generic SQL translator, automatic production ordering pass or unrelated
  correctness fix belongs here.

## Compatibility and conflict gates

Moving source statements changes patch contents/order and therefore the
persisted schema hash chain, even if the final database is identical.
The base identifies itself as `4.1-beta3` in `core/version/version.go`;
verify and record the applicable beta migration/upgrade policy before
editing. Do not promise existing-schema in-place compatibility or force
acceptance of changed hashes. If supported installations require the old
chain, stop and obtain a release-policy decision.

This task is independent of the mechanical schema-layout prerequisite.
It uses stock paths; reconcile moves with the layout owner when the changes
meet, rather than stacking on or modifying that owner's branch.

Several independently reviewed fixes touch definitions that may move:
nullable keys, exposure scope, lookup keys, `IIF`, Boolean handling and
other view changes. Preserve their ownership and reviewed semantics.
Record moved definitions for conflict resolution; integrate landed changes
by their actual revisions and regenerate, not by choosing entire files.
Do not import pending SQL fixes to make this cleanup pass.

The PostgreSQL harness must consume the resulting source order after
integration, reject new unresolved references with useful diagnostics,
and retire unnecessary relocation code separately. Its remaining H3
identity/clock/trigger work and other shared-source blockers are unchanged.
Do not modify the separately owned CI or combined-fix integration branches.

## Implementation and validation (2026-11-18)

The code commit splits early user, architecture, infrastructure and
workload definitions into dedicated source files, moves other table,
index and view definitions after their prerequisites, and reorders
definitions within files. It does not change SQL predicates or stored
types. Generated changelog and other triggers remain after the base DDL.
The source-order guard replays tables and indexes with foreign keys
enabled, checks view dependencies with SQLite's authorizer, and prepares
isolated triggers against all three write operations. Its fixtures cover
same-patch forward tables, keys, views and triggers, key collation,
composite keys and self-references. The guard itself uses SQLite even in
the dqlite-tagged test build.

- SQLite and real dqlite: race-enabled schema, schema-core, generator,
  model-import, export and recovery suites passed at the code head. The
  real-dqlite run used `make run-go-tests` and the shared musl/dqlite
  dependencies. Schema fresh creation, reapplication, trigger consumers,
  import/export round trips and recovery tests were included.
- A SQLite before/after comparison against `d380e5e4fb` matched 297
  controller schema objects and 95 tables, and 747 model schema objects
  and 251 tables, including seed rows. The comparison normalizes one
  trailing space removed from a moved `v_secret_permission` comment;
  that is the only observed schema-text difference. Its read-only helper
  is retained in this session's `files/compare-schema.go`.
- `make rebuild-export` produced no generated diff. Scoped
  `golangci-lint`, repository `txncheck`, `gci` and `git diff --check`
  passed.
- Race-enabled SQLite guard stress completed 15 runs with no failures
  before the intentional 21-second outer timeout (exit 124). This is
  bounded stress evidence, not an indefinitely passing run.
- Two focused dqlite invocations with a compound `TEST_FILTER` failed in
  the make recipe's shell quoting, before tests ran. The full dqlite
  schema suite was run successfully instead.
- Both cumulative post-patch tests skip because the registries are empty;
  that is not cumulative-upgrade evidence. Earlier beta databases cannot
  upgrade in place across changed hashes. Live 4.0 migration, PostgreSQL
  execution and the full repository pre-check were not run.
- The user reported running the live before/after 4.1 QA and added the
  observed dqlite schema diff to [the PR draft](pr.md). The reported diff
  shows the removed trailing space in the `v_secret_permission` comment;
  this session verified the captured diff in `/tmp/tmp.pWYfR7qIxu` but
  did not independently perform the bootstraps. The user accepted this
  schema-comparison QA as passed on 2026-11-18.
- Live teardown investigation, 2026-11-18: `schema-order-after` became
  stuck destroying; its `qa` model's machine 0 was down while LXD instance
  `juju-2482d2-0` remained running. Provisioner logs showed two overlapping
  attempts to start that machine, followed by repeated instance-exists
  errors. Its agent then failed to authenticate and exited. The model had
  three unforced machine-removal jobs for the same dying instance;
  removal retried while waiting for the instance to become dead. The
  connection between the overlapping provisioning and authentication
  failure is not established, nor is any relationship to schema ordering.
  The expected successful workload deployment and normal teardown are
  therefore not established by this QA run.
- With user approval, deleted the QA machine container and then forced
  destruction of `schema-order-after`. Verified both its machine and
  controller LXD instances and their identified profiles are absent, and
  the controller is no longer registered locally. The source worktrees,
  binaries and schema captures in `/tmp/tmp.pWYfR7qIxu` were left intact.

Commands for the final broad suites:

```sh
go test -race -p 2 -count=1 -timeout 120s \
  ./domain/schema/... ./core/database/schema \
  ./generate/triggergen ./generate/ddlgen \
  ./domain/modelimport/... ./domain/export/... ./domain/recovery/...

timeout 600s make run-go-tests \
  TEST_PACKAGES='./domain/schema/... ./core/database/schema ./generate/triggergen ./generate/ddlgen ./domain/modelimport/... ./domain/export/... ./domain/recovery/...' \
  TEST_ARGS='-race -p 2 -count=1 -timeout 120s'
```

## Original implementation and acceptance contract

1. Recheck upstream, reservations and applicable nested source guidance;
   confirm release-policy permission and record the before-state.
2. Inventory definitions, indexes, views, triggers and seed statements.
   Choose explicit file/statement moves that break the patch grouping
   cycles. No general solver is required by the observed object graph.
3. Add a regression guard that fails on a forward table reference, a key
   supplied too late, and a view defined before its source, including
   within one patch. Cover valid self-reference, ordinary prior references
   and supported multi-column keys. Keep the guard independent of
   PostgreSQL and avoid fragile keyword searches through comments/literals.
4. Preserve source SQL statement bytes where possible and compare full
   before/after schema objects, constraints, indexes, triggers and seeded
   data. Explain any unavoidable non-move diff. Verify write/trigger timing,
   not just the final object counts.
5. Exercise fresh creation, identical reapplication and all currently
   supported cumulative upgrades on SQLite and real dqlite. Cover required
   FK rejection/actions and representative view/trigger consumers. A
   skipped cumulative test with no registered patches is not upgrade
   evidence.
6. Run targeted race-enabled schema/generator checks with the branch's
   own APIs and shared `_deps`, then required bounded stress, formatting,
   scoped lint/txncheck and schema/static checks. PostgreSQL execution is
   not required for this SQLite/dqlite-only task.
7. Prepare a conventional `refactor(schema)` commit and local PR draft
   describing the final source-order invariant, hash-chain compatibility
   boundary and reproducible functional QA. Obtain separate approval
   before publishing anything.

Future targeted commands, not completed validation:

```sh
go test -race -p 2 -count=1 -timeout 120s \
    ./domain/schema/... ./core/database/schema \
    ./generate/triggergen ./generate/ddlgen

make run-go-tests \
    TEST_PACKAGES='./domain/schema/... ./core/database/schema ./generate/triggergen ./generate/ddlgen' \
    TEST_ARGS='-race -p 2 -count=1 -timeout 120s'
```

Use the [validation guide](../../.github/skills/upstream-breakout/references/validation.md)
for logging, real-dqlite dependencies, stress and static-analysis details.
Only the read-only SQLite dependency audit has run; none of the acceptance
checks above has been completed for a cleanup implementation.
