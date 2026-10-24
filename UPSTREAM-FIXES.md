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
| U4 | latent (test) | `domain/lease/state/migration_test.go` | Seeded leases with `date('now')` and discarded the `StdTxn` error, so a failed seed went unnoticed and the tests checked an empty table. | Fixed, Stage 6, `fix(lease): make lease pinning and tests valid on postgresql`. |
| U5 | latent (test) | `domain/changestream/state/state_test.go` `insertControllerNodes`; `domain/upgrade/state/state_test.go` | Insert integer `dqlite_node_id` values into the `TEXT` column, relying on SQLite type affinity. | changestream fixed, Stage 6, `test(changestream): insert controller node ids as text`; upgrade open. |
| U6 | latent (test) | `domain/model/state/testing/model.go` (lines 122, 240) | Uses `""` as an empty string literal. SQLite accepts double-quoted strings only as a legacy misfeature; standard SQL reads `""` as an identifier. | Open, from the Stage 5 findings. |

Reviewed and not a dqlite defect (recorded so they are not re-examined):

- `PinLease` ignored a unique violation inside its transaction. SQLite
  rolls back only the failing statement, so this was correct on dqlite;
  PostgreSQL aborts the whole transaction. Fixed as portability, in the
  same Stage 6 lease commit, with `ON CONFLICT DO NOTHING`.
- `TestSetAPIAddressesSharedPeerRollsBack` violates
  `controller_peer_address.controller_id NOT NULL` on purpose to force a
  rollback. The service never builds a shared peer projection. Only the
  SQLite-specific error text in the assertion was changed.
- Unqualified columns in `ON CONFLICT ... DO UPDATE` `SET` / `WHERE`
  clauses, and `ON CONFLICT DO UPDATE` with no conflict target
  (`domain/sequence`), are ambiguous or invalid only on PostgreSQL.
