# Upstream fix register

Concrete 4.1 fixes and shared portability improvements exposed by the PoC.
Their release value does not depend on finishing PostgreSQL support or HA.
Classes are defined in [PLAN.md](PLAN.md#session-protocol). "Fixed" below
means implemented in the PoC, not reviewed, independently extractable or
merged upstream. Record upstream PR/revision and validation when available.
[UPSTREAM-PLAN.md](UPSTREAM-PLAN.md) gives standalone stock-4.1 tasks.
This is the evolving record of findings, provenance and upstream status.
[UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md) contains only the reusable
procedure for bringing landed fixes back into the PoC. Neither is required
setup or QA for stock-upstream implementation agents.

## Keeping the register current

Record new findings here even when no implementation task exists yet.
For each finding, maintain the defect/rationale, affected paths, source
revision if available, upstream branch/PR/merge status, and eventual PoC
reconciliation revision/evidence. Distinguish found, implemented, reviewed,
landed and reintegrated; one does not imply the next.

When a finding is ready to assign, give UPSTREAM-PLAN.md a self-contained
upstream task describing the problem and required behaviour. Do not make
that task depend on this register's source commits or PoC test setup.
The integration procedure should not change merely because a new finding
or source mapping is added here.

## Extraction gate

For each proposed 4.1 change:

- Identify the dqlite defect or portability benefit and any behaviour delta.
- Isolate a coherent patch with no PostgreSQL-only/generated dependencies.
  U1/U2 share a PoC commit but are separate commits on the existing
  `4.1-sqlite-integers` review branch. U8/U9 include PG output to omit.
  Verify the extracted patch; do not assume PoC commits are self-contained.
- Carry a regression test and default/dqlite evidence on the target 4.1
  revision; preserve legacy import/export representations for schema fixes.
- Record the PoC commit, extraction adjustments, validation and upstream
  disposition (not proposed / in review / merged / deferred).

Existing entries have not been revalidated or assigned upstream PRs during
this plan refinement. Prioritise concrete bugs and schema-correctness fixes
for release review; do not wait for all portability suites to be green.


| ID | Class | Location | Defect on dqlite | Status |
|---|---|---|---|---|
| U1 | bug | `v_revision_updater_application_unit` (model schema) | Grouped by `u.uuid`, so it returned one row per unit instead of one unit count per application (D18). | Fixed, Stage 4, `fix(schema): correct lookup types and revision unit counts`. |
| U2 | latent | Model and controller schema, eleven lookup FK columns | Declared `TEXT` while referencing integer lookup IDs. SQLite coercion hid the mismatch, and exports carried the IDs as strings. | Fixed, Stage 4, same commit as U1. |
| U3 | latent | `internal/database/bootstrap.go` `InsertControllerNodeID` | Bound a `uint64` to the `TEXT` `dqlite_node_id` column. It worked only because the bootstrap node always uses go-dqlite's `BootstrapID` (high bit clear); `database/sql` rejects a `uint64` with the high bit set. | Fixed, Stage 6, `fix(database): bind controller node id as text`. |
| U4 | latent (test) | `domain/lease/state/migration_test.go` | Seeded leases with `date('now')` and discarded the `StdTxn` error, so a failed seed went unnoticed and the tests checked an empty table. | Fixed, Stage 6, `test(lease): seed migration test leases with bound timestamps`. |
| U5 | latent (test) | `domain/changestream/state/state_test.go` `insertControllerNodes`; `domain/upgrade/state/state_test.go` | Insert integer `dqlite_node_id` values into the `TEXT` column, relying on SQLite type affinity. | changestream fixed, Stage 6, `test(changestream): insert controller node ids as text`; upgrade open. |
| U6 | latent (test) | `domain/model/state/testing/model.go` (lines 122, 240) | Uses `""` as an empty string literal. SQLite accepts double-quoted strings only as a legacy misfeature; standard SQL reads `""` as an identifier. | Fixed, Stage 6, `test(domain): use single-quoted string literals in test sql` (49 files). |
| U7 | bug | `v_model_config` (model patch 0027) read by `domain/modelconfig/state` `ModelConfig` | If `model_config` also holds `agent-stream`, the view returns two `agent-stream` rows and `ModelConfig` keeps the last. SQLite's `UNION` sorts, so the `agent_version` stream wins only when it sorts after the config value (DIALECT.md X15). | Fixed, Stage 6, `fix(schema): prefer agent_version values in v_model_config`. The model branch excludes `agent-stream` and `agent-version` and the view uses `UNION ALL`; `TestModelConfigAgentStreamAndVersionInConfig` fails on dqlite without it. |
| U8 | latent | Controller schema: `user_activation_key.activation_key`, the eight `bakery_config` key columns, `macaroon_root_key.root_key` | Declared `TEXT` but bound as raw random bytes (activation key, bakery key pairs, `dbrootkeystore` root keys). SQLite stores them as BLOBs regardless; `domain/schema/storage.go` `binaryTextColumns` and recovery carried the mismatch, and exports typed them as strings. PostgreSQL rejects them (DIALECT.md X11), breaking `add-user`, the bootstrap worker's bakery initialisation and macaroon minting. | Fixed, Stage 6, `fix(schema): store random key bytes in blob columns`. The columns become `BLOB` in place; the 4.1 controller export types become `[]byte`. Upstream, drop the `domain/schema/postgresql/` paths from the commit. |
| U9 | latent | Model schema `removal.force` | Declared `INT` while every writer binds a Go bool or `FALSE` and every reader scans a bool. SQLite stores booleans as integers, so the mismatch was hidden, and exports carried the flag as an integer. PostgreSQL rejects a boolean for `bigint` (DIALECT.md X25). | Fixed, Stage 6, `fix(schema): declare the removal force flag as boolean`. `BOOLEAN NOT NULL DEFAULT FALSE` in place; the 4.1 export type becomes `bool` and the 4.0.12 import delta converts 0 / 1 and rejects other values. Upstream, drop the `domain/schema/postgresql/` paths from the commit. |
| U10 | portability | `domain/blockcommand/state` `GetBlocks` | Ordered by `rowid`, an implicit SQLite column that PostgreSQL lacks (DIALECT.md X31). On dqlite the order was insertion order. | Fixed, Stage 5, `fix(blockcommand): order blocks by type instead of rowid` (D31). dqlite order changes to type order; `TestGetBlocksOrderedByType` fails with `rowid`. |
| U11 | latent | `domain/crossmodelrelation/state/model/secrets.go` (two queries) | Group by `sruc.secret_id` but select `sr.uuid` and test `sruc.current_revision` in `HAVING`, so with several consuming units SQLite picks an arbitrary unit row (DIALECT.md X16). Also uses `instr` (X30). | Fixed upstream in review: juju/juju#23688 (open), branch `4.1-crossmodel-secret-revisions`. A secret is reported when any remote consuming unit is behind, and the initial query returns its latest revision once. Units are matched by an exact `<application>/` prefix. The companion commit removes bare grouped columns from the local consumed-secret queries in `domain/secret/state`. Both commits were cherry-picked unchanged onto `pg-poc/05-queries` on 2026-10-26 (`8f2c49c355` / `79f2e66049`, identical patch IDs), so the PoC is not blocked by review. When the PR lands, reconcile them under UPSTREAM-INTEGRATION.md. PostgreSQL at `08=c5f185e6f6`: `TestModelSecretsSuite` 29/29 and the two new watcher tests pass. Of the five consumed-secret `stateSuite` tests, two pass; three fail in fixture setup on separate `domain/secret/state` X16 sites (DIALECT.md). |
| U12 | latent | Model schema `relation_status_type.id`, `relation_status.relation_status_type_id` | Declared `TEXT` while the seed rows, `EncodeRelationStatus` and every writer use integers and the offer view compares with `1`. SQLite coercion hid it, and exports carried the ids as strings. pgx will not encode a Go int for `TEXT` (DIALECT.md X23). | Fixed, Stage 6, `fix(schema): declare relation status type ids as integers` (D30). 4.1 export types become `int64`; the 4.0.12 import delta parses the legacy strings. |
| U13 | bug | `domain/keymanager/state` `ensureUserPublicKey` | Read the new key id with `LastInsertId`. When the `INSERT ... SELECT` inserted nothing because the fingerprint hash algorithm was unknown, SQLite returned the connection's previous rowid, so the model could be linked to an unrelated key. | Fixed, Stage 5, `fix(keymanager): return new public key ids from the insert` (D36, DIALECT.md X35). An unknown algorithm returns an error; new test. |

Shared portability improvements on Stage 5. Review each for behaviour
deltas and extraction readiness; "portability" is not a guarantee of no
observable dqlite change (notably X36's reused-UUID error):

- X16: change stream latest-change query (`fix(changestream): select
  latest changes without bare columns`).
- X7: `IIF` in `GetModelConnectionInfo` (`fix(model): replace iif in the
  model connection info query`).
- X24: `x IN cte` (`fix(domain): select from ctes in in-list
  subqueries`).
- X18: ignored constraint errors inside transactions (`fix(domain): avoid
  ignoring constraint errors inside transactions`).
- X34: boolean flags compared with integers (`fix(operation): compare
  operation query flags with booleans`).
- X33: `octet_length` of non-text values (`fix(operation): measure pruned
  operation sizes as text`). Follow-up: a stored-bytes measure such as
  `pg_column_size` needs backend-aware state code.
- X36: several `ON CONFLICT` clauses (`fix(objectstore): use a single
  conflict clause when putting metadata`). A reused uuid now returns
  `ErrHashAndSizeAlreadyExists`.
- X37: SQLite's `==` in three queries (`fix: use standard sql equality in
  three domain queries`). `DeletePublicKeysForUser`'s unused-key cleanup
  had no test; one was added.
- X38: comma join scope in `GetDetachInfoForStorageAttachment`
  (`fix(removal): spell out the fulfilment cross join in detach info`).
  The fulfilment CTE is a one-row aggregate, so `CROSS JOIN` is the
  intended join, not a rewrite. A new test covers a fulfilment count of
  zero, which dqlite also lacked.
- Upstream: X37 and X38 are proposed together as `4.1-sql-standard-syntax`
  (UPSTREAM-PLAN.md), one commit each. Not yet extracted.

Known gap, not a dqlite defect: `constraint.allocate_public_ip` stays `INT`
in dqlite (`STRICT` table and the go-dqlite nullable-bool bug), and a
PostgreSQL-only override makes it `BOOLEAN` (DIALECT.md X25). The export
type is generated from the dqlite schema (`*int64`), so exporting a
PostgreSQL model would not match until export generation knows about
PostgreSQL overrides.

Reviewed and not a dqlite defect (recorded so they are not re-examined):

- `PinLease` ignored a unique violation inside its transaction. SQLite
  rolls back only the failing statement, so this was correct on dqlite;
  PostgreSQL aborts the whole transaction. Fixed as portability on
  Stage 5, `fix(lease): resolve lease pin conflicts in sql`, with
  `ON CONFLICT DO NOTHING` (DIALECT.md X18).
- `TestSetAPIAddressesSharedPeerRollsBack` violates
  `controller_peer_address.controller_id NOT NULL` on purpose to force a
  rollback. The service never builds a shared peer projection. Only the
  SQLite-specific error text in the assertion was changed.
- Unqualified columns in `ON CONFLICT ... DO UPDATE` `SET` / `WHERE`
  clauses, and `ON CONFLICT DO UPDATE` with no conflict target
  (`domain/sequence`), are ambiguous or invalid only on PostgreSQL
  (DIALECT.md X3, X4). Fixed as portability on Stage 5.

## Source provenance

Snapshot recorded during 2026-10-25 branch planning. These hashes identify
historical implementations, not current branch tips or proof of upstream
landing. Keep subsequent upstream/reintegration results with the finding.
Upstream tasks are specified by behaviour in UPSTREAM-PLAN.md, not by
requiring these commits to be available.

| Finding | Historical source / extraction |
|---|---|
| U1/U2 | PoC `367c35c243`; separate review commits `7c96d020d5` / `6b0f24d11c` on `4.1-sqlite-integers`. |
| U12 | `ad9d3928c1`; approved addition to the existing integer branch. |
| U8 | `e987e3df60` |
| U9 | `1513cf1b97` |
| U7 | `63f0bb3a68` |
| U13 | `0f7d28ec1d` |
| U3/U5 | `e8ef9d307b` / `03743f82ff` (the latter covers changestream fixtures). |
| U4 | `bae6008b6e` |
| U11 | Upstream `6068fa451a` / `4992247076` (juju/juju#23688, on `04bc64ef54`); PoC cherry-picks `8f2c49c355` / `79f2e66049`. |
| X16 change-stream query | `48f0c5e1a9` |
| U10/X31 | `a8301140de` |
| X34/X33 | `18886acfdf` / `e2258798fe` |
| X36 | `d658b402a6` |
| X37 / X38 | `0703122e2a` / `18d679a2b7` |
| U6/X6 | `8a60208176` |
| X3/X4, X24, model IIF | `ff382936b3`, `fda7add768`, `9499d95ca6` |
| X18 | `01316d98fe`, `bccb8d37b4` |

Source-specific cautions:

- `f474710187` mixes syntax, aggregation and lease timing changes. U4's
  `formatTimestamp` depends on it; that does not make the broad rewrite
  an upstream prerequisite. Use upstream-native helpers for the task.
- U1/U2 share a PoC commit but were split in their review branch; do not
  duplicate the unit-count fix while adding later integer corrections.
- U3 and the X16 latest-row source patches lacked dedicated regressions.
  Existing code is evidence, not proof of upstream acceptance.
- Cumulative generated files can contain other findings' changes. Inspect
  the assigned source rather than treating its entire tree as the fix.
