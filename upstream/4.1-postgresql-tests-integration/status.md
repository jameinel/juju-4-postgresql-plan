# 4.1-postgresql-tests-integration

[Shared direction](../../AGENTS.md) |
[Upstream dashboard](../../UPSTREAM-PLAN.md) |
[Integration plan and initial PR inventory](../4.1-postgresql-tests/integration-plan.md)

## Current state

Owner/session: `0ec926e6-3858-4f43-bc9c-658024127e88`, assigned by the user
on 2026-11-12. Local branch reserved at
`6e677c28505645413b55862e732c28e576ced3b9`, the initial infrastructure
configuration checkpoint. It has no combined-fix commits, separate worktree,
PostgreSQL validation or PR. This is a reservation, not the completed
demonstration. The user explicitly approved pushing this exact reservation
head on 2026-11-12. It was pushed atomically with the other three branches to
`jameinel/juju` and verified with `git ls-remote`. No PR creation or further
publication is authorized.

## Scope and next actions

Once infrastructure and CI exist, build the disposable demonstration from
their exact current heads plus selected pending upstream fixes. The linked
plan records 19 inspected open code/schema PR heads; refresh their heads,
dependency order and owner status before integration. Do not infer approval
or resolve the storage-attachment semantic hold from this inventory.

Preserve provenance, handle moved-schema conflicts explicitly, regenerate
PostgreSQL output from the combined source, and remove obsolete overrides.
Classify every failure without broad skips, assertion weakening or unrelated
PoC runtime imports. Unassigned shared schema findings need their own fixes.

Record the exact actual source manifest, full package coverage, backend
configuration, cleanup and complete-job timing evidence. Rebuild as upstream
fixes land. An eventual green combined run is not proof that the infrastructure
alone works against stock 4.1; rerun after the required fixes merge upstream.
Never merge this mixed demonstration branch wholesale into upstream.
