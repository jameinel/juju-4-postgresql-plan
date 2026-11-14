# X2: Juju empty-collection membership

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Retargeted with user approval on 2026-11-14 from a sqlair rendering
proposal to a Juju caller-level portability fix. Target repository:
[juju/juju](https://github.com/juju/juju), upstream `4.1`. Implementation
not started; no separate branch/worktree is reserved or required. Retain
the directory path as a stable task identifier, not a sqlair assignment.

Scheduled as [H3.1](../4.1-postgresql-tests/integration-plan.md#h31-juju-empty-collection-membership)
of explicit PostgreSQL test enablement. Inventory and SQLite/dqlite fixes
can proceed alongside H3 and H2.1; neither schema generation nor sqlair
placeholder adoption depends on this work. Complete the affected cases
before final H4 acceptance. H3.1 is a work checkpoint, not a mandatory PR
boundary: default to a separate reviewable commit in the PostgreSQL
test-enablement PR, outside the schema-generator commit.

SQLite defines empty-list membership behavior; this is a requirement of
adding PostgreSQL coverage, not evidence of an existing schema defect.
Extract a standalone correctness fix only if the caller audit finds
incorrect behavior on existing backends. If the portability changes are
substantial, a separate PostgreSQL-enablement prerequisite PR may help
reviewability, but independent schema-robustness value is not assumed.

- [ ] Inventory production and test membership queries across the
  SQL-owning coverage, not just domain state packages. Trace callers that
  can supply nil or empty collections; identify existing guards.
- [ ] Record each affected method, full predicate, input contract,
  required side effects and focused regression tests before changing it.
- [ ] Implement and validate caller-level handling in a separate
  PostgreSQL-enablement commit, then record its revision and any extracted
  correctness fixes needed by the integration run.

## Ownership and scope

SQLair expands `$S[:]` into a comma-separated placeholder list; it does
not own the surrounding membership predicate. Its deliberate empty/nil
slice contract predates the placeholder work (canonical/sqlair#136).
Juju owns the operation's empty-input policy and must avoid sending
`IN ()` / `NOT IN ()`, which SQLite accepts and PostgreSQL rejects.
Classify this as portability, not a demonstrated dqlite behavior bug.

Use caller guards or explicit query variants that preserve each method's
contract. Empty input may mean no matches, no filter, or an error; do not
invent one policy for all callers. For actual empty-set membership,
`x IN empty` is false and `x NOT IN empty` is true, even for NULL `x`.
Preserve nil-versus-empty result shapes, validation/errors, other
predicates, aggregates, required transaction work and side effects.
Returning early is valid only where the complete operation permits it.
Test write paths explicitly so an empty collection cannot accidentally
broaden a mutation.

This supersedes the proposed generic `SELECT NULL WHERE FALSE` expansion
in sqlair. Do not substitute `IN (NULL)`, change generic slice expansion,
add a driver rewrite, or introduce PostgreSQL-only arrays/SQL. Richer
SQLair membership support would be a separate future API proposal,
not a prerequisite for this task or canonical/sqlair#195.

## Acceptance and reconciliation

- Cover nil, empty and nonempty inputs; `IN` and `NOT IN`; NULL operands
  or list members where supported; mixed lists and composite predicates
  where present. Assert the method's results/errors and side effects,
  not merely successful query execution.
- Validate the caller changes on SQLite and real dqlite, with race/stress
  where required. Their existing-backend regression tests must not require
  PostgreSQL, generated PostgreSQL assets or a new sqlair release.
- Before final affected H4 acceptance, run the regressions on PostgreSQL
  through the native H2.1 sqlair path and H3 schemas, with no empty-`IN`
  rewrite, added skips or weakened assertions. Record exact integrated
  heads; a combined pass does not establish upstream landing.
- After landing, the PoC integration owner reconciles the caller fixes
  and removes the Stage 3 X2 rewrite only after its consumers are covered.
  Do not remove it merely because a sqlair placeholder release exists.

Stop for ambiguous caller semantics or an unrelated behavior change.
No implementation, complete caller inventory or validation is claimed yet.
