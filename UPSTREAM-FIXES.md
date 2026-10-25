# Upstream fix register

These are dqlite-side defects that the PostgreSQL work exposed. Each one
should be fixed in Juju whatever happens to the PoC. Classes are as in
[PLAN.md](PLAN.md#session-protocol), session protocol step 6. A fixed
entry names the commit holding the fix, which can be proposed upstream on
its own.

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
| U11 | latent | `domain/crossmodelrelation/state/model/secrets.go` (two queries) | Group by `sruc.secret_id` but select `sr.uuid` and test `sruc.current_revision` in `HAVING`, so with several consuming units SQLite picks an arbitrary unit row (DIALECT.md X16). Also uses `instr` (X30). | Open (D35). Portable design in `handoffs/2026-10-25-stage-6-test-sql.md` section 6. |
| U12 | latent | Model schema `relation_status_type.id`, `relation_status.relation_status_type_id` | Declared `TEXT` while the seed rows, `EncodeRelationStatus` and every writer use integers and the offer view compares with `1`. SQLite coercion hid it, and exports carried the ids as strings. pgx will not encode a Go int for `TEXT` (DIALECT.md X23). | Fixed, Stage 6, `fix(schema): declare relation status type ids as integers` (D30). 4.1 export types become `int64`; the 4.0.12 import delta parses the legacy strings. |
| U13 | bug | `domain/keymanager/state` `ensureUserPublicKey` | Read the new key id with `LastInsertId`. When the `INSERT ... SELECT` inserted nothing because the fingerprint hash algorithm was unknown, SQLite returned the connection's previous rowid, so the model could be linked to an unrelated key. | Fixed, Stage 5, `fix(keymanager): return new public key ids from the insert` (D36, DIALECT.md X35). An unknown algorithm returns an error; new test. |

Portability cleanups (valid on dqlite, invalid on PostgreSQL; no dqlite
behaviour change). Each is on Stage 5 and can be proposed upstream on its
own:

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
- Open: X37 (`==` in three queries) and X38 (comma join scope in removal
  storage), see DIALECT.md.

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
  Stage 5, `fix(lease): 
resolve lease pin conflicts in sql`, with
  `ON CONFLICT DO NOTHING` (DIALECT.md X18).
- `TestSetAPIAddressesSharedPeerRollsBack` violates
  `controller_peer_address.controller_id NOT NULL` on purpose to force a
  rollback. The service never builds a shared peer projection. Only the
  SQLite-specific error text in the assertion was changed.
- Unqualified columns in `ON CONFLICT ... DO UPDATE` `SET` / `WHERE`
  clauses, and `ON CONFLICT DO UPDATE` with no conflict target
  (`domain/sequence`), are ambiguous or invalid only on PostgreSQL
  (DIALECT.md X3, X4). Fixed as portability on Stage 5.
