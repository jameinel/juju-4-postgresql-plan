# PostgreSQL PoC: handoff, Stage 8 cleanup tool (2026-10-27)

> Snapshot for the next session. PLAN.md, the stage files and DECISIONS.md
> stay authoritative; where they disagree with this note, they win.

Goal: finish Stage 8 with 08-cleanup-tool, then hand over to Stage 9.
Follow the PLAN.md session protocol:

- Ask the user before any design choice, one question at a time.
- Make one commit per concern on the owning stage branch, then push.
- Never run two PostgreSQL test packages at once, and never print the
  database URL.
- Leave the user's untracked `PR-4.1-*.md` files in the plan repo alone.

## 1. Where things are

The code tree is clean on `pg-poc/08-dbaccessor` at `fc8f6f7f88`. Every
`pg-poc/*` branch is pushed; 08 is the top of the stack.

08-delete is done ([result](../stages/08-dbaccessor.md#08-delete-result)):
`postgresql.DeleteSchema` drops a namespace's schema with RESTRICT (D47),
and `pgWorker.DeleteDB` calls it through `PostgresNamespaces.Delete`. The
delete transaction uses READ COMMITTED, an approved exception to D7
recorded in D47 (DIALECT.md X40).

Still open from earlier sessions: U14 has no UPSTREAM-PLAN.md task, and
D46 is still provisional.

## 2. Next: 08-cleanup-tool

A separate operator script, `scripts/postgresql-poc/drop-juju-schemas.py`,
so that repeated PoC bootstraps can reuse one PostgreSQL database (D23:
bootstrap fails fast if the controller or controller model schema
already exists). Required checks (stage file task table):

- a non-destructive inventory;
- an explicit target and confirmation;
- a controller-stopped precondition;
- never a blanket cleanup on bootstrap failure.

Facts to start from:

- Schemas are named `juju_<namespace>` (`postgresql.DefaultSchemaPrefix`,
  `Placement.SchemaName`). Test schemas start with `juju-test-`
  (`pgtesting.SchemaNamePrefix`) and must not be matched. In SQL, `_` is
  a `LIKE` wildcard, so `LIKE 'juju_%'` matches them too; escape it.
- `scripts/postgresql-poc/deploy-postgresql.py` (on
  `pg-poc/01-provision`) writes the `db-url` file and runs external
  commands with `subprocess`; it does not use a Python PostgreSQL
  driver.
- `DeleteSchema` refuses objects Juju did not create. A cleanup tool for
  a whole controller may need CASCADE; that is a design choice for the
  user, as is the way to detect a running controller.

Ask the user, one at a time, before choosing: the branch that owns the
script (the task is in Stage 8, the provisioning script is on Stage 1),
how the script connects (`psql` or a driver), what the inventory shows,
RESTRICT or CASCADE, and how the controller-stopped precondition is
checked.

After 08-cleanup-tool, the stage file's acceptance section hands over to
Stage 9: real bootstrap, added-model creation, restart and completed
deletion on both PoC backends.
