# Upstream fix register

Concrete 4.1 fixes and shared portability improvements exposed by the PoC.
Their release value does not depend on finishing PostgreSQL support or HA.
Classes are defined in [PLAN.md](PLAN.md#session-protocol). "Fixed" below
means implemented in the PoC, not reviewed, independently extractable or
merged upstream. Record upstream PR/revision and validation when available.
[UPSTREAM-PLAN.md](UPSTREAM-PLAN.md) gives standalone remaining stock-4.1
tasks. [UPSTREAM-DONE.md](UPSTREAM-DONE.md) records implemented upstream
work, decisions and rationale, including work still in review.
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
After implementation, move execution details and decision rationale to
UPSTREAM-DONE.md; retain outstanding actions in UPSTREAM-PLAN.md and update
this register's finding/status/provenance links.
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

Upstream PRs are listed under
[Upstream review status](#upstream-review-status). The Status column below
records the PoC implementation. Prioritise concrete bugs and
schema-correctness fixes for release review; do not wait for all
portability suites to be green.


| ID | Class | Location | Defect on dqlite | Status |
|---|---|---|---|---|
| U1 | bug | `v_revision_updater_application_unit` (model schema) | Grouped by `u.uuid`, so it returned one row per unit instead of one unit count per application (D18). | Fixed, Stage 4, `fix(schema): correct lookup types and revision unit counts`. |
| U2 | latent | Model and controller schema, eleven lookup FK columns | Declared `TEXT` while referencing integer lookup IDs. SQLite coercion hid the mismatch, and exports carried the IDs as strings. | Fixed, Stage 4, same commit as U1. |
| U3 | latent | `internal/database/bootstrap.go` `InsertControllerNodeID` | Bound a `uint64` to the `TEXT` `dqlite_node_id` column. It worked only because the bootstrap node always uses go-dqlite's `BootstrapID` (high bit clear); `database/sql` rejects a `uint64` with the high bit set. | Fixed, Stage 6, `fix(database): bind controller node id as text`. |
| U4 | latent (test) | `domain/lease/state/migration_test.go` | Seeded leases with `date('now')` and discarded the `StdTxn` error, so a failed seed went unnoticed and the tests checked an empty table. | Fixed, Stage 6, `test(lease): seed migration test leases with bound timestamps`. |
| U5 | latent (test) | `domain/changestream/state/state_test.go` `insertControllerNodes`; `domain/upgrade/state/state_test.go` | Insert integer `dqlite_node_id` values into the `TEXT` column, relying on SQLite type affinity. | changestream fixed, Stage 6, `test(changestream): insert controller node ids as text`; upgrade open in the PoC, fixed upstream in juju/juju#23685. |
| U6 | latent (test) | `domain/model/state/testing/model.go` (lines 122, 240) | Uses `""` as an empty string literal. SQLite accepts double-quoted strings only as a legacy misfeature; standard SQL reads `""` as an identifier. | Fixed, Stage 6, `test(domain): use single-quoted string literals in test sql` (49 files). |
| U7 | bug | `v_model_config` (model patch 0027) read by `domain/modelconfig/state` `ModelConfig` | If `model_config` also holds `agent-stream`, the view returns two `agent-stream` rows and `ModelConfig` keeps the last. SQLite's `UNION` sorts, so the `agent_version` stream wins only when it sorts after the config value (DIALECT.md X15). | Fixed, Stage 6, `fix(schema): prefer agent_version values in v_model_config`. The model branch excludes `agent-stream` and `agent-version` and the view uses `UNION ALL`; `TestModelConfigAgentStreamAndVersionInConfig` fails on dqlite without it. |
| U8 | latent | Controller schema: `user_activation_key.activation_key`, the eight `bakery_config` key columns, `macaroon_root_key.root_key` | Declared `TEXT` but bound as raw random bytes (activation key, bakery key pairs, `dbrootkeystore` root keys). SQLite stores them as BLOBs regardless; `domain/schema/storage.go` `binaryTextColumns` and recovery carried the mismatch, and exports typed them as strings. PostgreSQL rejects them (DIALECT.md X11), breaking `add-user`, the bootstrap worker's bakery initialisation and macaroon minting. | Fixed, Stage 6, `fix(schema): store random key bytes in blob columns`. The columns become `BLOB` in place; the 4.1 controller export types become `[]byte`. Upstream, drop the `domain/schema/postgresql/` paths from the commit. |
| U9 | latent | Model schema `removal.force` | Declared `INT` while every writer binds a Go bool or `FALSE` and every reader scans a bool. SQLite stores booleans as integers, so the mismatch was hidden, and exports carried the flag as an integer. PostgreSQL rejects a boolean for `bigint` (DIALECT.md X25). | Fixed, Stage 6, `fix(schema): declare the removal force flag as boolean`. `BOOLEAN NOT NULL DEFAULT FALSE` in place; the 4.1 export type becomes `bool` and the 4.0.12 import delta converts 0 / 1 and rejects other values. Upstream, drop the `domain/schema/postgresql/` paths from the commit. |
| U10 | portability | `domain/blockcommand/state` `GetBlocks` | Ordered by `rowid`, an implicit SQLite column that PostgreSQL lacks (DIALECT.md X31). On dqlite the order was insertion order. | Fixed, Stage 5, `fix(blockcommand): order blocks by type instead of rowid` (D31). dqlite order changes to type order; `TestGetBlocksOrderedByType` fails with `rowid`. |
| U11 | latent | `domain/crossmodelrelation/state/model/secrets.go` (two queries) | Group by `sruc.secret_id` but select `sr.uuid` and test `sruc.current_revision` in `HAVING`, so with several consuming units SQLite picks an arbitrary unit row (DIALECT.md X16). Also uses `instr` (X30). | Implemented upstream on open [PR #23688](https://github.com/juju/juju/pull/23688), branch `4.1-crossmodel-secret-revisions`, head `aa6cad849c`. A secret is reported if any remote consuming unit is behind, and the initial query returns its latest revision once through indexed offer connections. Tracking uniqueness/upserts use `(offer_connection_uuid, secret_id, unit_name)`; synthetic-unit cleanup uses one CTE-backed DELETE. The independent local-consumer fix, [PR #23720](https://github.com/juju/juju/pull/23720) (`4.1-local-secret-revisions`, head `ee07d7ccb5`), merged 2026-11-04 as `edb2aef5b9`; review regressions and redundant-DISTINCT removal are complete. Earlier versions were cherry-picked onto `pg-poc/05-queries` on 2026-10-26 (`8f2c49c355` / `79f2e66049`, matching patch IDs then). Reconcile landed #23720 and later #23688 independently under UPSTREAM-INTEGRATION.md; current upstream follow-ups are not established as integrated. Historical PostgreSQL evidence at `08=c5f185e6f6`: `TestModelSecretsSuite` 29/29 and two new watcher tests pass; two of five local consumed-secret tests pass, while three fail in fixtures on separate secret-state X16 sites. |
| U12 | latent | Model schema `relation_status_type.id`, `relation_status.relation_status_type_id` | Declared `TEXT` while the seed rows, `EncodeRelationStatus` and every writer use integers and the offer view compares with `1`. SQLite coercion hid it, and exports carried the ids as strings. pgx will not encode a Go int for `TEXT` (DIALECT.md X23). | Fixed, Stage 6, `fix(schema): declare relation status type ids as integers` (D30). 4.1 export types become `int64`; the 4.0.12 import delta parses the legacy strings. |
| U13 | bug | `domain/keymanager/state` `ensureUserPublicKey` | Read the new key id with `LastInsertId`. When the `INSERT ... SELECT` inserted nothing because the fingerprint hash algorithm was unknown, SQLite returned the connection's previous rowid, so the model could be linked to an unrelated key. | Fixed, Stage 5, `fix(keymanager): return new public key ids from the insert` (D36, DIALECT.md X35). An unknown algorithm returns an error; new test. |
| U14 | bug | `internal/worker/dbaccessor/tracker.go` | Leaked database pools. `openDatabase` did not close the pool when setting the foreign keys pragma failed; `newTrackedDBWorker` did not close it when model initialisation failed; `ensureDBAliveAndOpenIfRequired` never closed replacement pools it opened during reopen attempts that then failed their ping. Each leak kept a pool open until the process exited. | Fixed in PoC Stage 8 at `b6a7690d80` `fix(dbaccessor): close databases the tracker opens but does not keep`. Historical PoC validation: four new tests fail without the fix; race-enabled stress 214 runs, 0 failures. The tracker files at the original `pg-poc/08` base matched upstream `04bc64ef54`. [PR #23726](https://github.com/juju/juju/pull/23726) (`4.1-dbaccessor-pool-cleanup`, head `ee55bab5f4`) merged 2026-11-04 as `5259edff43`. Reconciliation of the landed patch against the existing PoC fix remains pending. |
| U15 | latent | Model schema tables with a nullable primary key column: `application_exposed_endpoint_cidr` and `_space` (`application_endpoint_uuid`, NULL is the wildcard endpoint), `charm_container_mount.charm_container_key`, `charm_manifest_base` (`os_id`, `track`, `branch`, `architecture_id`). | SQLite allows NULL in a non-integer primary key and treats NULLs as distinct, so wildcard rows insert, but the key does not stop duplicates of them. PostgreSQL makes every primary key column NOT NULL (DIALECT.md X44). **Found in Stage 9 (2026-10-28):** the first PostgreSQL bootstrap failed in `upsertExposedCIDRs` (`domain/application/state/exposed.go`) exposing the controller application with 23502. Only the exposure tables store NULL. | **Implemented upstream first, not yet in the PoC:** the initial split was published on 2026-10-30 from upstream `8369e6e2a9`. Current heads are charm `838abf41ea` in [#23710](https://github.com/juju/juju/pull/23710) and exposure `4e66f7c449` in [#23723](https://github.com/juju/juju/pull/23723); both remain open, and exposure must be rebased/regenerated after charm lands. Exposure on all endpoints moves to application-wide `application_exposed_space` / `_cidr` tables, mirroring bindings, and endpoint tables require the endpoint; the view still reports wildcard rows with a NULL endpoint. Mount container keys become `NOT NULL`; manifest bases use `(charm_uuid, array_index, nested_array_index)` and reject duplicate legacy positions with an identifying diagnostic. Exposure owns the combined schema guard. Legacy deltas split exposure scope and deduplicate wildcard rows; space removal also avoids alpha-duplicate UNIQUE failures. |
| U16 | latent | `domain/agentbinary/state/{controller,model}/state.go` `RegisterAgentBinary` | On a primary key conflict it runs a follow-up `SELECT` in the same transaction. SQLite keeps the transaction usable after the failed statement; PostgreSQL does not (DIALECT.md X18, 25P02 "current transaction is aborted"). The caller's `AlreadyExists` / `AgentBinaryImmutable` handling is never reached. **Found in Stage 9 (2026-10-28):** retries after the U15 failure re-register the binary and fail with 25P02. | Implemented upstream on open [PR #23728](https://github.com/juju/juju/pull/23728), branch `4.1-agentbinary-conflict`, head `6a66e5ed1b`. Both paths use `ON CONFLICT (version, architecture_id) DO NOTHING` and compare the retained registration, preserving `AlreadyExists` / `AgentBinaryImmutable`, idempotency and transaction usability. Not integrated into the PoC; its bootstrap blocker remains. |
| U17 | latent | 48 model and 17 controller single-column `id INT PRIMARY KEY` tables at `8369e6e2a9` | SQLite permits NULL keys; exported `*int64` IDs allow malformed payload rows to reach persistence. | Implemented on independent branch `4.1-lookup-key-not-null`, [PR #23724](https://github.com/juju/juju/pull/23724), head `4e7341392f`; open for approval. Declares `INT PRIMARY KEY NOT NULL`, regenerates scalar IDs, validates payload/recovery boundaries and cross-checks decoder inventory against DDL. Explicit typed deltas and the 65-key scope are retained. No PoC integration is recorded. |
| U18 | tooling | `Makefile` `install-dependencies`; static-analysis prerequisites | `install-dependencies` covers Go snap and `libsqlite3-dev`, but not `shellcheck`, `gci`, `golangci-lint` or `stress`. Without `shellcheck`, `STATIC_ANALYSIS=1 make pre-check` stops at dependency checks; without SQLite headers it fails schema generation with `sqlite3.h` missing. The run also reports skipped `go-licenses`, `shfmt` and `govulncheck` checks when those tools are absent. | Found 2026-11-03; not implemented. Independently inventory required and skipped development tools, add appropriate installation/verification to `make install-dependencies`, and validate clean and repeated installs plus the affected checks. See `4.1-install-dependencies` in UPSTREAM-PLAN.md. |

### U17: nullable INT lookup primary keys (independent PR in review)

**Class:** latent. **Status:** implemented as independent PR
[juju/juju#23724](https://github.com/juju/juju/pull/23724), based on refreshed
upstream 4.1 rather than stacked on either U15 PR. The published follow-up
at `4e7341392f` adds schema-derived decoder-inventory checks, direct
`NotValid` classification and the transaction-test fix. The proposed
generic helper was tried and reverted; retain explicit typed deltas.
Remaining work is approval/CI and later combined-guard reconciliation.

At upstream `8369e6e2a9`, 65 tables have a single-column `id INT PRIMARY KEY`
with no explicit NOT NULL: **48 model, 17 controller**. The inventory was
obtained from that ref's CREATE TABLE declarations in an in-memory SQLite
database and `pragma_table_info`, not merely a textual count of INT tokens.
INTEGER PRIMARY KEY is a different SQLite type spelling: it aliases rowid;
INT PRIMARY KEY does not. SQLite accepts NULL INT keys and duplicate
NULL-keyed rows, while standard SQL makes primary keys NOT NULL.

`generate/export/main.go:sqliteTypeToGoType` maps INT to int64 and adds a
pointer when the column is nullable, so these keys export as `*int64`.
Making them `INT PRIMARY KEY NOT NULL` gives `int64` through the existing
generator; changing the generator to infer NOT NULL from every primary key
would hide rather than repair the storage declaration.

Ordinary seed rows and `domain/cloud/state.AllowCloudType` already write
integer IDs. Payload import/recovery are alternative producers: nullable
legacy IDs can reach the database without this fix. It rejects absent IDs
rather than inventing zero (a real seed ID). For 4.0.12 model lookup types,
generated struct-shape changes require typed deltas; preserve
all fields and validate consumed rows before dereferencing. Controller
export types start at 4.1.0, so no legacy controller delta is required.
Source changestream lookup rows are excluded by model import and must remain
excluded; they are not a reason to reintroduce source namespace IDs.

Implemented representation safeguards:

- `decodeLookupPayload` validates raw YAML before scalar decoding loses
  NULL/omitted versus zero. A DDL-derived inventory test ties its lists to
  the 65 changed keys, with changelog exclusions and the seven previously
  non-NULL keys recorded as deliberate exceptions.
- Recovery validates column presence and NULL versus scalar fields;
  model/controller decode/replay and failure atomicity have regressions.
- Explicit deltas retain full rows, nullable non-ID fields and valid zero,
  while rejecting absent IDs with table/row context and `NotValid`.
  Seeded-import conflict behaviour and target-local changelog exclusions
  remain covered. Recorded live 4.0-to-4.1 and 4.1-to-4.1 migrations passed;
  that is not PoC reconciliation evidence.

**No semantic dependency on U15:** only shared schema/generated files and
integration of the schema guard overlap. Use an INT-key-scoped schema guard
on the independent branch, not the U15 whole-schema guard (which would fail
while unrelated charm/exposure keys remain nullable). If the exposure guard
has landed, narrow its exception to single-column INTEGER keys with the
lookup fix. Otherwise tighten it when the independent changes are combined,
regenerate and validate both guards. U15's two branches and PoC bootstrap
requirements remain unchanged. No PoC integration or PostgreSQL experiment
is recorded for U17. The standalone scope and acceptance checks are in
[UPSTREAM-PLAN.md](UPSTREAM-PLAN.md#lookup-key-not-null-task-separate-session).

Inventory at the inspected base (all affected columns are `id`):

- Model: `agent_stream`, `architecture`, `block_command_type`,
  `block_device_provenance`, `change_log_edit_type`, `change_log_namespace`,
  `charm_config_type`, `charm_provenance`, `charm_relation_role`,
  `charm_relation_scope`, `charm_resource_kind`, `charm_run_as_kind`,
  `charm_source`, `charm_storage_kind`, `container_type`, `hash_kind`,
  `ip_address_config_type`, `ip_address_origin`, `ip_address_scope`,
  `ip_address_type`, `k8s_pod_status_value`, `life`, `link_layer_device_type`,
  `machine_cloud_instance_status_value`, `machine_placement_scope`,
  `machine_status_value`, `network_address_scope`,
  `operation_task_status_value`, `os`, `password_hash_algorithm`, `protocol`,
  `resolve_mode`, `resource_origin_type`, `resource_retrieved_by_type`,
  `resource_state`, `secret_grant_scope_type`, `secret_grant_subject_type`,
  `secret_role`, `secret_rotate_policy`, `ssh_key_algorithm_type`,
  `storage_filesystem_status_value`, `storage_kind`, `storage_provision_scope`,
  `storage_volume_device_type`, `storage_volume_status_value`,
  `unit_agent_status_value`, `virtual_port_type`, `workload_status_value`.
- Controller: `architecture`, `auth_type`, `change_log_edit_type`,
  `change_log_namespace`, `cloud_type`, `lease_type`, `life`, `model_type`,
  `object_store_drain_phase_type`, `password_hash_algorithm`,
  `permission_access_type`, `permission_object_access`,
  `permission_object_type`, `secret_backend_type`,
  `ssh_fingerprint_hash_algorithm`, `ssh_key_algorithm_type`,
  `upgrade_state_type`.

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
  operation sizes as text`). Upstream #23693 also fixes NULL-column sums:
  NULL contributes zero rather than making the whole row ignored. That
  changes the estimate and is not reconciled into the PoC. A stored-bytes
  measure such as `pg_column_size` remains separate backend-aware work.
- X36: several `ON CONFLICT` clauses (`fix(objectstore): use a single
  conflict clause when putting metadata`). In the PoC a reused uuid
  returns `ErrHashAndSizeAlreadyExists`. The published review
  follow-up for juju/juju#23694 returns the documented
  `ErrMetadataUUIDAlreadyExists` sentinel for different hashes or size,
  preserving the initial PR's error text. It must remain distinct from
  `ErrHashAndSizeAlreadyExists`, which callers treat as "already stored".
  Both service put methods preserve the sentinel through wrapping; state
  tests cover hash and size-only UUID conflicts and unchanged paths/hints.
  Follow-up `53feca2259` landed in PR #23694 on 2026-10-30 (merge commit
  `75ddfcbaf0b3`). Reconcile the PoC's UUID-reuse classification separately.
- X37: SQLite's `==` in three queries (`fix: use standard sql equality in
  three domain queries`). `DeletePublicKeysForUser`'s unused-key cleanup
  had no test; one was added.
- X38: comma join scope in `GetDetachInfoForStorageAttachment`
  (`fix(removal): spell out the fulfilment cross join in detach info`).
  The fulfilment CTE is a one-row aggregate, so `CROSS JOIN` is the
  intended join, not a rewrite. A new test covers a fulfilment count of
  zero, which dqlite also lacked.
- Upstream: X37 and X38 are in juju/juju#23695, one commit each. A third
  commit uses `=` in the storage lifecycle triggers. That changes the
  trigger patch's schema hash, so a model database created by an earlier
  4.1 beta fails the hash check. The released 4.0.15 DDL is unchanged.

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
  `ON CONFLICT DO NOTHING` (DIALECT.md X18). The independent upstream
  implementation is in open PR #23729.
- `TestSetAPIAddressesSharedPeerRollsBack` violates
  `controller_peer_address.controller_id NOT NULL` on purpose to force a
  rollback. The service never builds a shared peer projection. Only the
  SQLite-specific error text in the assertion was changed.
- Unqualified columns in `ON CONFLICT ... DO UPDATE` `SET` / `WHERE`
  clauses, and `ON CONFLICT DO UPDATE` with no conflict target
  (`domain/sequence`), are ambiguous or invalid only on PostgreSQL
  (DIALECT.md X3, X4). Fixed as portability on Stage 5; the independent
  upstream sequence fix is in open PR #23731.

## Upstream review status

Snapshot refreshed from GitHub on 2026-11-04: 9 open and 14 merged PRs.
Every listed PR targets `juju/juju` branch `4.1` from `jameinel/juju`.
Head hashes are short SHA
prefixes; check GitHub before acting because review and CI status can change.

| Branch | Findings | PR | Head | Status |
|---|---|---|---|---|
| `4.1-sqlite-integers` | U1, U2, U12 | [#23680](https://github.com/juju/juju/pull/23680) | `0e9be8ddd7` | merged 2026-10-30; merge `8369e6e2a9` |
| `4.1-binary-key-columns` | U8 | [#23682](https://github.com/juju/juju/pull/23682) | `de69a748a6` | merged 2026-10-27; merge `fa1743d544` |
| `4.1-model-config-precedence` | U7 | [#23683](https://github.com/juju/juju/pull/23683) | `f50dfe4257` | open; CI green; approval required |
| `4.1-keymanager-insert-id` | U13 | [#23684](https://github.com/juju/juju/pull/23684) | `d490585ad7` | merged 2026-10-27; merge `f0adf0150d` |
| `4.1-controller-node-id-text` | U3, U5 | [#23685](https://github.com/juju/juju/pull/23685) | `55ee33f048` | merged 2026-10-27; merge `6e98deb0e0` |
| `4.1-lease-migration-tests` | U4 | [#23686](https://github.com/juju/juju/pull/23686) | `7e3e903650` | merged 2026-10-27; merge `146df46e46` |
| `4.1-crossmodel-secret-revisions` | U11 | [#23688](https://github.com/juju/juju/pull/23688) | `aa6cad849c` | open; mergeable; refreshed-head CI pending; approval required |
| `4.1-removal-force-boolean` | U9 | [#23690](https://github.com/juju/juju/pull/23690) | `f39572e43b` | merged 2026-10-27; merge `9ea82c03bd` |
| `4.1-changestream-latest-row` | X16 | [#23691](https://github.com/juju/juju/pull/23691) | `9e28053164` | merged 2026-10-30; merge `ece31c289c` |
| `4.1-blockcommand-order` | U10/X31 | [#23692](https://github.com/juju/juju/pull/23692) | `495768b0aa` | merged 2026-10-27; merge `b36f88d99c` |
| `4.1-operation-query-portability` | X34, X33 | [#23693](https://github.com/juju/juju/pull/23693) | `0fb2eb7194` | open; CLA/commit/static checks pass; default-bases smoke fails, macOS cancelled; review required |
| `4.1-objectstore-conflicts` | X36 | [#23694](https://github.com/juju/juju/pull/23694) | `53feca2259` | merged 2026-10-30; merge `75ddfcbaf0` |
| `4.1-sql-standard-syntax` | X37, X38 | [#23695](https://github.com/juju/juju/pull/23695) | `13003d8680` | merged 2026-10-27; merge `302b44cc2c` |
| `4.1-sql-test-literals` | U6 | [#23696](https://github.com/juju/juju/pull/23696) | `11ff0981e0` | merged 2026-10-27; merge `1696e6128d` |
| `4.1-nullable-primary-keys` | U15 (charm keys) | [#23710](https://github.com/juju/juju/pull/23710) | `838abf41ea` | open; macOS cancelled/aggregate CI fails; approval required; land before exposure |
| `4.1-local-secret-revisions` | U11 follow-up | [#23720](https://github.com/juju/juju/pull/23720) | `ee07d7ccb5` | merged 2026-11-04; merge `edb2aef5b9`; PoC reconciliation pending |
| `4.1-application-exposure-scope` | U15 (exposure + combined guard) | [#23723](https://github.com/juju/juju/pull/23723) | `4e66f7c449` | open; bundles smoke fails; approval required; rebase/regenerate after charm lands |
| `4.1-lookup-key-not-null` | U17 | [#23724](https://github.com/juju/juju/pull/23724) | `4e7341392f` | open; CI green; approval required; independent of U15 |
| `4.1-dbaccessor-pool-cleanup` | U14 | [juju/juju#23726](https://github.com/juju/juju/pull/23726) | `ee55bab5f4` | merged 2026-11-04; merge `5259edff43`; PoC reconciliation pending |
| `4.1-agentbinary-conflict` | U16/X18 | [#23728](https://github.com/juju/juju/pull/23728) | `6a66e5ed1b` | open; default-bases smoke fails; approval required; not integrated into PoC |
| `4.1-lease-pin-conflict` | X18 | [#23729](https://github.com/juju/juju/pull/23729) | `c55d5839e5` | open; static check fixed; bundles/revision smoke fail; approval required |
| `4.1-dbrepl-latest-row` | X16 | [#23730](https://github.com/juju/juju/pull/23730) | `49a1bdf808` | merged 2026-11-04; merge `f7b596bf5c` |
| `4.1-sequence-upsert-target` | X3/X4 | [#23731](https://github.com/juju/juju/pull/23731) | `74aabf93b4` | open; test/comment follow-up published 2026-11-04; new-head CI/review unassessed; allocator refactor still uncommitted |
| `4.1-drop-iif` | X7 | [published branch](https://github.com/jameinel/juju/tree/4.1-drop-iif); no PR | `08b9cc0d7c` plus uncommitted DDL | query published 2026-11-04; finish schema-view validation and revise draft before further publication approval |
| `4.1-cte-membership-subqueries` | X24 | [published branch](https://github.com/jameinel/juju/tree/4.1-cte-membership-subqueries); no PR | `8ce0410f51` | implementation published 2026-11-04; historically validated; upstream PR approval pending |
| `4.1-install-dependencies` | U18 | not proposed | | not implemented |
| `4.1-secret-latest-revision` | X16 | not proposed | | not implemented; follows remaining #23688 review |

Notes:

- The PR descriptions are drafted in `PR-4.1-<branch>.md`.
- No unresolved inline review threads were reported for the nine open PRs;
  all still require approval. CI states above are a revision-specific
  snapshot, not evidence of PoC acceptance or a diagnosis of failing jobs.
- juju/juju#23730 completes the separate dbrepl latest-row SQL task. No PoC
  integration was performed; dbrepl remains outside the PostgreSQL PoC
  scope. The duplicated output-helper refactor remains a separate candidate
  in UPSTREAM-PLAN.md. Historical old/new SQLite/Dqlite checks and the
  corrected REPL-only scope are preserved in
  [the execution evidence](UPSTREAM-DONE.md#dbrepl-latest-row-selection-x16).
- juju/juju#23720 and #23726 merged on 2026-11-04. Compare their landed
  patches with the existing PoC implementations under UPSTREAM-INTEGRATION.md;
  earlier cherry-picks or equivalent fixes do not establish reconciliation.
  U14's independent upstream implementation, nine regressions and package
  race/stress results are preserved in
  [the pool-cleanup evidence](UPSTREAM-DONE.md#dbaccessor-pool-cleanup-u14).
  Retain the later PostgreSQL tracker/backend adaptations when comparing
  with PoC `b6a7690d80`.
- juju/juju#23685 also fixes the upgrade test half of U5.
- juju/juju#23696 also fixes double-quoted string literals in production
  SQL, not only in tests.
- X7 and X24 have published fork branches but no upstream PR;
  continue those implementations through the tracking links in UPSTREAM-PLAN.md.
  X18 is represented by the
  U16 agent-binary review and the separate `PinLease` review above.
- U16, `PinLease` and sequence allocation already have independent review
  branches; continue those rather than starting new implementations.
  Developer tooling and the five-query latest-secret rewrite remain new
  implementation tasks in UPSTREAM-PLAN.md.
- U15 was written upstream first, with no PoC source commit. The published
  split keeps charm keys in juju/juju#23710 and exposure scope in stacked
  juju/juju#23723. Both are needed before the next PostgreSQL bootstrap;
  charm alone leaves the wildcard failure intact. Integrate both under D43
  and record the PoC commits here; no PoC reintegration has occurred.
  Validation/publication details below are historical and revision-specific;
  they do not establish coverage of the current branch heads.

### Cross-machine branch reconciliation (2026-11-04)

Verified the local Juju branches/worktrees against live
`jameinel/juju` refs and the `juju/juju` PR heads. The PR table above
records published heads, not necessarily the branch checked out here.
Local presence, publication, landing and PoC reconciliation remain distinct.
The 2026-11-04 CI/review snapshot above was retained from the remote
planning update; this reconciliation rechecked heads and merge state,
not every check or review thread.

#### Pre-cleanup inventory

| Branch | Local head | Published head | Local state / action |
|---|---|---|---|
| `4.1-drop-iif` | `08b9cc0d7c` | absent | Query committed; three controller-view replacements uncommitted in `domain/schema/controller/sql/0007-model.sql`. Continue here, not on a new branch. Draft currently excludes those views and needs revision. |
| `4.1-cte-membership-subqueries` | `8ce0410f51` | absent | Clean worktree; eleven expressions in eight production files and focused regressions implemented. Publication approval pending. |
| `4.1-sequence-upsert-target` | `74aabf93b4` | `cfa8676090` | One unpublished test follow-up plus uncommitted common-allocator/RETURNING refactor and tests in `domain/sequence/state/{sequence.go,sequence_test.go}`. Do not overwrite with the published head or treat PR CI as local validation. |
| `4.1-model-config-precedence` | `3c27c2371e` | `f50dfe4257` | Clean worktree at a different head; compare before updating it. |
| `4.1-crossmodel-secret-revisions` | `375ac766cb` | `aa6cad849c` | Clean worktree at a different head; compare before updating it. |
| `4.1-operation-query-portability` | `e463565601` | `0fb2eb7194` | Clean worktree at a different head; compare before updating it. |
| `4.1-nullable-primary-keys` | `ba21f48ad3` | `838abf41ea` | Clean worktree at a different head; compare before updating it. |
| `4.1-local-secret-revisions` | `9eaa8d7b03` | `ee07d7ccb5` | Clean worktree; upstream PR merged. Do not assume the local branch includes the landed result. |
| `4.1-lookup-key-not-null` | `3157cdb288` | `4e7341392f` | Clean worktree at a different head; compare before updating it. |
| `4.1-lease-pin-conflict` | `373ab4f422` | `c55d5839e5` | Clean worktree at a different head; published retry-safe test-helper follow-up is not established locally. |
| `pg-poc/00-base-fixes` through `pg-poc/08-dbaccessor` | matches published refs | matches local refs | All nine stage tips agree. Stage 08 is `950bba743e`. |
| `pg-poc/09-bootstrap` | `950bba743e` | absent | Local branch exists at the stage-08 tip, with no unique commit. Do not infer stage-09 implementation or a published branch. |

`4.1-agentbinary-conflict`, `4.1-application-exposure-scope`,
`4.1-dbaccessor-pool-cleanup` and `4.1-dbrepl-latest-row` have clean
worktrees matching their published heads. The main Juju checkout is clean
at local `4.1` = `e22f5666d0`, not a freshly updated upstream baseline.
This inventory was captured before the user-requested cleanup below.

Planning history diverged after `068c9c2`: local `caa46c5` contained task
claims and execution evidence, while remote `4d636d8` contained later
landing/review/tooling updates. Both histories are retained by a local
merge, with pre-merge local tip at
`backup/main-before-reconcile-20261104`. Five previously untracked PR
texts are retained unchanged in this repository: agentbinary, pool cleanup,
dbrepl, drop-iif and lease-pin. They are historical/draft text, not a new
publication approval. The CTE/sequence descriptions and sequence review
reply were also recovered from the original session artifacts into
`PR-4.1-cte-membership-subqueries.md`,
`PR-4.1-sequence-upsert-target.md` and
`COMMENT-4.1-sequence-upsert-review.md`; historical/stale sequence metadata
is labelled explicitly. These local files can travel with the planning
repository instead of depending on one machine's session storage.

#### Transfer artifacts and cleanup

The goal is to continue on the other machine without losing near-complete
local work. On 2026-11-04, removed all 13 clean linked Juju worktrees after
rechecking for tracked and untracked changes. Retained the main `../juju`
checkout and the dirty `../4.1-drop-iif` and
`../4.1-sequence-upsert-target` worktrees unchanged. Local implementation
branches remain, including CTE; only the explicitly aborted
`pg-poc/09-bootstrap` branch was deleted after verifying it equalled stage
08 and had no unique commits. Its clean PoC worktree was also removed.

The following artifacts travel with this planning repository:

| Artifact under `handoffs/` | Preserved work |
|---|---|
| `2026-11-04-local-work.bundle` | Exact branch tips for `4.1-drop-iif` (`08b9cc0d7c`), `4.1-cte-membership-subqueries` (`8ce0410f51`) and `4.1-sequence-upsert-target` (`74aabf93b4`). Four commits in total, including the published sequence implementation and unpublished test follow-up. |
| `2026-11-04-drop-iif-wip.patch` | Uncommitted replacements in the three controller schema views, based on `08b9cc0d7c`. |
| `2026-11-04-sequence-wip.patch` | Uncommitted common-allocator/RETURNING refactor and tests, based on `74aabf93b4`. |

`git bundle verify` passed. Both patches were applied to temporary indexes
at their recorded branch tips and reconstructed all three modified files
byte-for-byte, without changing either source worktree or its real index.
This verifies transfer integrity, not code/test correctness. Final IIF-view
validation and sequence-refactor validation remain outstanding.

The incremental bundle requires upstream commit
`e22f5666d0861d3c84e8465c1ff86fbd23f58f48` in the receiving Juju repository;
fetch upstream `4.1` first if necessary. From that repository, set `plan`
to the absolute path of this planning checkout, then import into new
transfer refs, not over the other machine's current branches:

```sh
plan=/absolute/path/to/juju-4-postgresql-plan
git --no-pager bundle verify "$plan/handoffs/2026-11-04-local-work.bundle"
git --no-pager fetch "$plan/handoffs/2026-11-04-local-work.bundle" \
  'refs/heads/*:refs/heads/transfer/20261104/*'
git --no-pager worktree add ../transfer-drop-iif \
  transfer/20261104/4.1-drop-iif
git --no-pager -C ../transfer-drop-iif apply --check \
  "$plan/handoffs/2026-11-04-drop-iif-wip.patch"
git --no-pager -C ../transfer-drop-iif apply \
  "$plan/handoffs/2026-11-04-drop-iif-wip.patch"
git --no-pager worktree add ../transfer-sequence \
  transfer/20261104/4.1-sequence-upsert-target
git --no-pager -C ../transfer-sequence apply --check \
  "$plan/handoffs/2026-11-04-sequence-wip.patch"
git --no-pager -C ../transfer-sequence apply \
  "$plan/handoffs/2026-11-04-sequence-wip.patch"
```

CTE is committed entirely on `transfer/20261104/4.1-cte-membership-subqueries`;
it needs no WIP patch. Compare these snapshots with the other machine's
current work before cherry-picking or incorporating changes. Do not reset
existing branches to these older bases. The local drafts above preserve
the scope, rationale and historical validation needed to finish each task.
No implementation branch or planning changes were pushed during cleanup.

#### Committed-branch publication after cleanup

On 2026-11-04, published the exact committed IIF and CTE snapshots as
`jameinel/juju:4.1-drop-iif` (`08b9cc0d7c`) and
`jameinel/juju:4.1-cte-membership-subqueries` (`8ce0410f51`), and
fast-forwarded `4.1-sequence-upsert-target` / juju/juju#23731 from
`cfa8676090` to `74aabf93b4`. Verified the remote refs.
Tracking links and remaining actions are in
[UPSTREAM-PLAN.md](UPSTREAM-PLAN.md#existing-implementations-and-published-tracking-references).
No uncommitted code, PR descriptions or review responses were published.
The existing transfer bundle and patches remain valid snapshots.

The other active 4.1/PoC branch commits are already reachable through fetched
remote refs, except the superseded operation commits `820aa88f88` and
`e463565601`. Those are already public in GitHub's history; the current
operation PR head is `0fb2eb7194`. Do not overwrite that newer branch with
the obsolete local tip. Historical backup and unrelated 2.9/3.6 experiment
branches are not publication targets for this handoff.

### Nullable primary-key base refresh (2026-10-30)

Refreshed upstream and the local `4.1` base to `8369e6e2a9` (merged
integer-schema PR #23680). Rebased charm commits to `7001a1b74b` and
`ba21f48ad3`; restacked exposure to `4d9518117c` and `f31de64e52`.
Both branches retain only their assigned concern. Original split heads are
preserved in `backup/charm-before-base-refresh-20261030` and
`backup/exposure-before-base-refresh-20261030`. Upstream integer conversions,
full-row tests and schema changes were retained, not replaced by older
generated files. The exposure transform conflict was resolved by generation
from the combined schema; regeneration is idempotent on both branches.

Verification on the refreshed revisions:

- Base and final stack each passed
  `timeout 600s make go-build GOFLAGS=-p=2` with existing `_deps`, building
  all six linux/amd64 agent/client targets. Dependencies were not installed.
- Charm: `make run-go-tests` with
  `TEST_PACKAGES="./generate/export ./domain/schema ./domain/modelimport/... ./domain/recovery/..."`
  and `TEST_ARGS="-race -count=1 -p=2 -timeout=8m"` passed all nine packages.
  Focused charm import stress: 132 completed runs, zero failures before
  the intentional 31-second timeout.
- Final exposure: `make run-go-tests` with
  `TEST_PACKAGES="./generate/export ./domain/schema ./domain/application ./domain/application/service ./domain/modelimport/... ./domain/recovery/... ./domain/network/state ./domain/removal/state/model"`,
  `TEST_ARGS="-race -count=1 -p=2 -timeout=12m"` and `TEST_FILTER=Test`
  passed all 13 packages. The first redirected attempt had no package-result
  evidence; the explicit Test selector rerun above established coverage.
- Exposure state: `TEST_PACKAGES="./domain/application/state"`,
  `TEST_ARGS="-race -count=1 -p=2 -timeout=4m"`,
  `TEST_FILTER=TestExposedStateSuite` passed (10.5 seconds).
  Exposure watcher stress: 6 completed runs, zero failures before the
  intentional 31-second timeout. Temporary stress binaries were removed.
- Both `make pre-check` invocations passed but skipped static analysis;
  explicit `golangci-lint run --build-tags=dqlite,libsqlite3` over the affected
  package sets reported zero issues. Non-generated Go files were checked
  with gci and the worktrees are clean.
- Scoped diff whitespace checks pass. Full branch `git diff --check` reports
  two new generated `WHEN ` lines inheriting the existing trigger template's
  trailing space. They were left generator-consistent, not hand-edited or
  used to expand this PR into a trigger-formatting refactor.
- IDE build validation succeeds for the open base project, with limited
  diagnostics; the unopened exposure/charm worktrees rely on actual Go
  compilation/tests and the successful binary builds. Live-controller QA,
  PostgreSQL tests and the entire application-state suite were not rerun.

Publication completed after explicit approval of all content/actions:

- Atomically force-updated `jameinel/juju:4.1-nullable-primary-keys` from
  `43eed70758` to `ba21f48ad3` using an explicit lease; pushed the new
  `4.1-application-exposure-scope` branch at `f31de64e52`.
- Replaced the title/body of
  [juju/juju#23710](https://github.com/juju/juju/pull/23710) with the reviewed
  charm-only description.
- Created [juju/juju#23723](https://github.com/juju/juju/pull/23723), not draft,
  targeting `4.1` with the reviewed stacked exposure description.
- Posted the [contextual reply](https://github.com/juju/juju/pull/23710#issuecomment-6527752522),
  restricted by a sub-agent to the review's three suggestions and actual
  changes. It does not discuss the independent lookup-key plan.
- Read both PRs and the comment back and verified exact approved bodies,
  titles, branch heads, base `4.1` and non-draft status. New-head CI has not
  been assessed. No review threads were resolved or plan changes pushed.

Further published edits require fresh explicit approval.

### Nullable primary-key split validation (2026-10-30, pre-refresh)

Approved local split: charm-only `4df04695d8` for juju/juju#23710, exposure
`3c90a7ded3` stacked on it. This initial extraction retained base
`9ea82c03bd`; it was later refreshed and verified as recorded above.
These historical results do not establish coverage of the refreshed head.
No branches, PR content or plan changes have been pushed.

Charm validation:

```sh
make rebuild-export
make run-go-tests \
  TEST_PACKAGES="./generate/export ./domain/modelimport ./domain/modelimport/transformer/transforms/to_v4_1_0 ./domain/schema ./domain/recovery/state" \
  TEST_ARGS="-race -count=1 -p 2 -timeout 8m" TEST_FILTER=Test
# Generator test initially failed to compile (wrong helper name); fixed.
make run-go-tests TEST_PACKAGES="./generate/export" \
  TEST_ARGS="-race -count=1 -p 2 -timeout 3m" TEST_FILTER=TestGeneratorSuite
# Pass; the other packages in the earlier command all passed.
make pre-check
# Pass, but STATIC_ANALYSIS is disabled by default.
golangci-lint run ./generate/export/... ./domain/modelimport/...
# Zero issues.
```

Exposure validation before restack: complete race suites passed for
`./domain/schema`, `./domain/application`, `./domain/application/service`,
`./domain/network/state`, `./domain/removal/state/model`,
`./domain/modelimport`, and `./domain/recovery/state` using
`make run-go-tests TEST_ARGS="-race -count=1 -p 2 -timeout 8m"`.

After resolving handwritten conflicts and regenerating from the combined
schema, these checks passed:

```sh
make rebuild-export
make run-go-tests \
  TEST_PACKAGES="./generate/export ./domain/modelimport/... ./domain/recovery/..." \
  TEST_ARGS="-race -count=1 -p 2 -timeout 8m" TEST_FILTER=Test
make run-go-tests TEST_PACKAGES="./domain/application/state" \
  TEST_ARGS="-race -count=1 -p 2 -timeout 3m" \
  TEST_FILTER=TestExposedStateSuite
make pre-check
golangci-lint run ./generate/export/... ./domain/application/... \
  ./domain/modelimport/... ./domain/network/state/... \
  ./domain/removal/state/model/... ./domain/schema/...
git --no-pager diff --check
```

The first combined post-restack command also selected the entire
application-state package. That package reached its cumulative 8-minute
timeout while `TestUnitStateSuite/TestGetUnitWorkloadVersion` was running;
its full suite is **not** verified green. The focused exposure-state suite
passed afterwards (27.7 seconds), including failure injection for all four
exposure tables and rollback assertions.

Race binaries compiled with `make run-go-tests -c` were stressed using
`timeout 31s stress -p 2 -timeout 30s`: charm import selector
`TestImportSuite/.*Charm` reported 12 completed runs, zero failures;
exposure watcher selector `TestWatcherSuite/TestWatchApplicationExposed`
reported 6 completed runs, zero failures. Exit 124 is the intentional
outer timeout. Session-local binaries were removed afterwards.

Formatting used `gci` on touched non-generated Go files. IDE build calls
could not target these unopened worktrees; successful Go race tests supply
compilation evidence instead. Existing `_deps` were reused after the
initial missing-musl attempt was stopped; no dependency manifest changed.
The final stacked exposure tree also passed
`timeout 600s make go-build GOFLAGS=-p=2` using the existing `_deps`:
`jujuc`, `containeragent`, `pebble`, `jujuagentd`, `juju` and
`juju-metadata` built successfully for linux/amd64. Live-controller QA
and PostgreSQL tests were not run.

### Integer-schema review follow-up (2026-10-29)

U1/U2/U12, juju/juju#23680: stronger conversion/view tests, `NotValid` errors
and developer schema-evolution documentation; merged on 2026-10-30 at head
`0e9be8ddd7` (merge `8369e6e2a9`). No PoC reintegration.
Race-enabled tests, 52 stress iterations and targeted lint passed; live QA
not run. [PR draft](PR-4.1-sqlite-integers.md) and
[review decisions/evidence](COMMENT-4.1-sqlite-integers-review.md) hold details.

### Separate agent-workflow guidance

`4.1-pr-description-guidance`: description/comment distinction, unrelated
to SQL fixes. [Patch](4.1-pr-description-guidance.patch) and
[PR draft](PR-4.1-pr-description-guidance.md) staged; no branch or PR yet.

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
| U15 | Upstream only. Original combined `43eed70758` is historical and backed up. Initial split: charm `4df04695d8`, exposure `3c90a7ded3` on `9ea82c03bd`. Published refreshed split on `8369e6e2a9`: charm `7001a1b74b` + `ba21f48ad3` in juju/juju#23710, exposure `4d9518117c` + `f31de64e52` in juju/juju#23723. No PoC reintegration. |
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
