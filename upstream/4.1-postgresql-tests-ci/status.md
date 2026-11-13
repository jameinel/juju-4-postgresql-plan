# 4.1-postgresql-tests-ci

[Shared direction](../../AGENTS.md) |
[Upstream dashboard](../../UPSTREAM-PLAN.md) |
[Integration plan](../4.1-postgresql-tests/integration-plan.md)

## Current state

Owner/session: `0ec926e6-3858-4f43-bc9c-658024127e88`, assigned by the user
on 2026-11-12. Local branch reserved at
`6e677c28505645413b55862e732c28e576ced3b9`, the initial infrastructure
configuration checkpoint. No CI-specific commits, worktree or PR exist yet.
The user explicitly approved pushing this exact reservation head on 2026-11-12.
It was pushed atomically with the other three branches to `jameinel/juju` and
verified with `git ls-remote`. No PR creation or further publication is
authorized.

## Scope and next actions

After infrastructure is functional, stack this branch on its updated head.
Provision a pinned disposable PostgreSQL service, wait for readiness, supply
the centrally handled backend/connection settings, run the complete SQL-owning
package inventory, retain safe diagnostics and clean up deterministically.
Keep existing SQLite/dqlite jobs intact.

Record the image, server settings, runner resources, selected packages and
complete cold/warm job timings. Acceptance is under 30 minutes, preferably
under 15, including provisioning, compilation, execution and cleanup.
Service version/settings are still implementation decisions, not approved
by the ref reservation. No PostgreSQL job has been run.

This branch does not contain shared SQL/schema fixes or PostgreSQL controller
runtime support. Demonstration against pending fixes belongs on the separate
integration branch. Land CI after infrastructure and the required fixes;
do not make an untrustworthy red check required.
