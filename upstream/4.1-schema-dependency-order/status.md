# 4.1-schema-dependency-order

[Shared direction](../../AGENTS.md) |
[Upstream dashboard](../../UPSTREAM-PLAN.md) |
[PostgreSQL test infrastructure](../4.1-postgresql-tests/status.md)

## Current state

Prepared 2026-11-18 by session
`5eb9907f-1835-4e64-98a2-d54e0aa2461d`, with the user's authorization
to split source ordering into an independent UPSTREAM task. Reservation
covers task preparation; implementation has not started. The user authorized
handoff to a separate thread on 2026-11-18. The receiving session may take
over this reserved branch/worktree and should record its session ID here
and in the dashboard before editing; no further permission from the
preparing session is required.

Verified upstream 4.1 base:
`d380e5e4fba4c9918a0774638ffa3138aeece73e`, checked through GitHub on
2026-11-18. Created branch/worktree: `4.1-schema-dependency-order`,
`../4.1-schema-dependency-order`. HEAD equals that base and the worktree is
clean, with `_deps -> ../../_deps` sharing the existing dependencies.
No cleanup commit, push or PR exists for this task. Task preparation is
recorded in the local planning checkout; nothing has been published.

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

## Implementation and acceptance

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
