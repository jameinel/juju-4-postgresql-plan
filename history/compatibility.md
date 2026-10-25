# Compatibility implementation history

Historical inventory through the 2026-10-25 Stage 6 run. Not a task list or
proof of the current tree. Current work lives in Stages 5/6; defect status
lives in UPSTREAM-FIXES.md and dialect handling in DIALECT.md. Git history
is authoritative for current commit placement after restacking.

## Stage 5 inventory

| Commit subject | Reference |
|---|---|
| `fix(domain): make shared sql queries valid on sqlite and postgresql` | X7, X8 |
| `fix(lease): resolve lease pin conflicts in sql` | X18 |
| `fix(domain): qualify on conflict upserts for postgresql` | X3, X4 |
| `fix(changestream): select latest changes without bare columns` | X16, D24 |
| `fix(status): read allocate_public_ip as a nullable bool` | X25, D29 |
| `fix(domain): select from ctes in in-list subqueries` | X24 |
| `fix(domain): avoid ignoring constraint errors inside transactions` | X18 |
| `fix(model): replace iif in the model connection info query` | X7, D32 |
| `fix(blockcommand): order blocks by type instead of rowid` | X31, U10, D31 |
| `fix(operation): compare operation query flags with booleans` | X34, D34 |
| `fix(operation): measure pruned operation sizes as text` | X33, D33 |
| `fix(keymanager): return new public key ids from the insert` | X35, U13, D36 |
| `fix(objectstore): use a single conflict clause when putting metadata` | X36, D37 |

The early changes were tested on dqlite only. Later changes were found and
checked with the PostgreSQL harness. Do not reuse an early "nothing run
on PostgreSQL" statement as current status.

## Stage 6 inventory

| Commit subject | Reference |
|---|---|
| `feat(testing): add PostgreSQL domain test harness` | N6, N9 |
| `fix(database): bind controller node id as text` | U3 |
| `test(lease): seed migration test leases with bound timestamps` | U4 |
| `test(schema): add backend-aware failing trigger helper` | portability |
| `test(controllernode): make address projection tests backend neutral` | portability |
| `test(changestream): insert controller node ids as text` | U5 |
| `fix(schema): store random key bytes in blob columns` | U8, X11 |
| `fix(schema): prefer agent_version values in v_model_config` | U7, X15 |
| `fix(schema): declare the removal force flag as boolean` | U9, X25 |
| `test(database): skip txn query logging test on postgresql` | D25 |
| `test(schema): seed controller api port as text` | X23 |
| `test(domain): quote the user table in test sql` | X8 |
| `test(domain): use single-quoted string literals in test sql` | X6, U6 |
| `test(domain): replace sqlite date functions in test sql` | X7 |
| `fix(schema): declare relation status type ids as integers` | U12, D30 |
| `test(crossmodelrelation): match secret consumer units without instr` | X30; not the U11 production fix |
| `fix(database): move schema dialects to a leaf package` | D11 |
| `test(domain): replace insert or ignore and replace in test sql` | X7 |

The environment-isolation fix was folded into the harness commit. The
explicit change-log ID fix (D13) was folded into Stage 4's generator commit.
Lease pinning belongs on Stage 5, not Stage 6.

## Lessons retained for future diagnosis

- `OsEnvSuite` cleared `JUJU_POSTGRESQL_TEST_URL`, so some supposedly PG
  suites ran on SQLite. The harness preserves it; repeat backend proof
  after changing environment isolation.
- The initial identity default consumed sequence values before the
  change-log trigger took its lock. The replacement uses a plain BIGINT
  and an owned sequence, preserving explicit fixture IDs (D13).
- PostgreSQL forbids subqueries in trigger `WHEN`; `FailingTriggerSQL`
  evaluates such conditions in the PL/pgSQL body. Failure-injection tests
  still need the same operation and rollback behaviour on both backends.
- The harness needed schema dialect information, but importing
  `core/database/schema` caused an import cycle. D11 moves that interface
  into a leaf package; do not reintroduce the dependency.

## Historical runs and restacking

The final handoff records 06 at `eac393da16`, 05 at `d658b402a6`, and 08 at
`2e67edc999`, with a clean tree and pushed branches. Build success on 08
is not runtime-worker verification. The recorded results are carried into
[Stage 6 detail](../stages/06-compatibility.md).

Logs from that run were under
`~/.copilot/session-state/5465307a-a217-4f78-8ff1-cc3b87cf91d7/files/`;
`restack/` was the latest group and `x23/` contained the older status run.
Earlier helper scripts were in session
`e1b8ccab-eefe-4bb9-a26f-d0c9097f66da/files/`. Use OPERATIONS.md to reproduce
runs; these private artifacts may not be available in a later session.

One earlier restack left 07/08 with copies of old 06 commits. Recovery
rebased 06 onto 05, then rebased the top with `--onto <new-06> <old-06>`.
Do not blindly repeat old hashes: inspect ancestry, preserve branch tips,
and verify every adjacent stage before choosing a recovery command.
