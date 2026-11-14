# sqlair-empty-in

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Proposed 2026-11-14. Unclaimed; this directory is a task identifier, not
evidence that a branch or worktree exists. Target repository:
[canonical/sqlair](https://github.com/canonical/sqlair). Independent of
the placeholder branches; schedule after them to avoid conflicts in
`internal/expr/querybuilder.go`.

- [ ] Confirm the rendering on SQLite, dqlite and PostgreSQL.
- [ ] Inventory Juju callers that can pass empty slices (UPSTREAM-PLAN.md
  X2 row).

## Scope

An empty slice input renders as `x IN ()` (PR #136 made empty and nil
slices valid). SQLite accepts it; PostgreSQL rejects it, and the Juju PoC
driver rewrites it (DIALECT.md X2). Render an empty slice portably so that
`x IN (<empty>)` is false and `x NOT IN (<empty>)` is true, including for
NULL `x`. Candidate: replace the empty placeholder list with
`SELECT NULL WHERE FALSE`, giving `x IN (SELECT NULL WHERE FALSE)`.
sqlair does not write the parentheses (input-expression docs), so check
other contexts where `$S[:]` may appear.

Removes the PoC X2 driver rewrite after a sqlair release.
