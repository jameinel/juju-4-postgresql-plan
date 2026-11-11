# 4.1-lookup-key-not-null

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Reported checks have no failures or pending jobs; obtain approval. Schema-derived decoder inventory, direct `NotValid` classification and the transaction-test fix are implemented. Retain explicit typed deltas and the 65-key scope; later reconcile the combined guard with exposure.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-lookup-key-not-null`, #23724 | U17 | Explicitly require INT lookup keys and reject absent payload IDs rather than silently treating them as valid zero IDs.

From `UPSTREAM-FIXES.md`:

`4.1-lookup-key-not-null` | U17 | [#23724](https://github.com/juju/juju/pull/23724) | `4e7341392f` | open; no failing/pending reported checks; approval required; independent of U15

### Upstream register snapshot

U17 | latent | 48 model and 17 controller single-column `id INT PRIMARY KEY` tables at `8369e6e2a9` | SQLite permits NULL keys; exported `*int64` IDs allow malformed payload rows to reach persistence. | Implemented on independent branch `4.1-lookup-key-not-null`, [PR #23724](https://github.com/juju/juju/pull/23724), head `4e7341392f`; open for approval. Declares `INT PRIMARY KEY NOT NULL`, regenerates scalar IDs, validates payload/recovery boundaries and cross-checks decoder inventory against DDL. Explicit typed deltas and the 65-key scope are retained. No PoC integration is recorded.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

## Lookup-key NOT NULL task (separate session)

Continue existing independent #23724, not a new implementation branch.
Scope: single-column `id INT PRIMARY KEY NOT NULL` (48 model, 17 controller
at `8369e6e2a9`), generated scalar IDs and consumed payload boundaries.
Recheck inventory at the actual base; leave INTEGER rowid keys, composite
keys, historical DDL, 4.0.12 export types and unrelated nullable fields alone.
Do not replace explicit schema constraints with generator inference.

Implementation, regression coverage and recorded live migration QA are in
UPSTREAM-DONE.md. Remaining work is review/CI and later combined-guard
reconciliation. Preserve this acceptance contract on further changes:

- Keep the explicit typed-delta methods: a generic loop still needs an ID
  accessor and field-mapping closure in every method, obscuring each
  conversion. `requiredLookupID` already centralises NULL rejection.
  Exercise every consumed legacy lookup type's full row, multiple rows,
  nullable non-ID fields, nil IDs and valid zero. Reject missing IDs with
  table/row context and invalid-payload classification; never invent zero.
  No legacy controller delta is needed.
- In `domain/export/payload.go`, validate consumed 4.1.0 model/controller
  raw YAML IDs before scalar decoding loses NULL/omitted versus zero.
  Test valid zero/non-zero, NULL and missing IDs at the actual boundary;
  do not broaden validation to unrelated scalar fields.
- Preserve import exclusions for `change_log_edit_type` and
  `change_log_namespace`, valid seed IDs and seeded-import conflict handling.
  Test representative legacy decode -> transform -> import -> export
  persistence, including existing seeds and a valid non-seed where allowed.
- Verify fresh model/controller schemas reject NULL keys and the complete
  affected declaration/generated-field set is correct. Exercise generated
  recovery decode/replay, valid dump round trips, NULL/missing rejection
  and failure atomicity without relaxing immutable-table requirements.
- Run affected schema/export/import/recovery/cloud-state race suites,
  focused stress/lint/build and regeneration-idempotence checks. Keep the
  INT-key guard independent; reconcile the global guard as described above.
  Stop for release-policy changes, new nullable-key writers or incompatible
  representations. No PostgreSQL-only work belongs in this review.

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Lookup-key NOT NULL decisions

The initial #23724 implementation (`3157cdb288`) was independent on
refreshed `8369e6e2a9`, not stacked on U15. At that base, SQLite
CREATE TABLE/`pragma_table_info` inventory found 48 model and 17 controller
single-column `id INT PRIMARY
KEY` declarations permitting NULL. INT is not the rowid-aliasing INTEGER
spelling, and duplicate NULL lookup keys were possible.

Explicit NOT NULL lets the existing export generator produce `int64`
instead of `*int64`; a blanket generator inference would hide the storage
defect. Only those keys and generated consumers change. Zero remains a
valid seed ID; absent/NULL IDs must not become zero during YAML decoding.
Consumed 4.0.12 model rows need typed deltas preserving every field and
validating before dereference; 4.0.12 types and historical DDL remain
unchanged. Controller exports start at 4.1.0, so no legacy controller delta
is required. Source changestream lookup rows remain excluded from import.
Recovery validates presence/NULL against scalar fields; model/controller
decode/replay and persistence regressions cover the regenerated types.

The published follow-up at `4e7341392f` adds a DDL-derived decoder-inventory
cross-check, direct `coreerrors.NotValid` classification, and the schema
guard's assertion outside its transaction. The proposed generic helper was
tried and reverted: each table still needed typed ID-accessor and mapping
closures, while `requiredLookupID` already centralizes NULL rejection.
Retain explicit deltas. The seven previously non-NULL keys are deliberate
inventory exceptions, not an expansion of this 65-key fix.
The [review response](https://github.com/juju/juju/pull/23724#issuecomment-6618410549)
records successful live 4.0-to-4.1 and 4.1-to-4.1 migrations. That does not
establish PoC integration. Remaining review/CI and combined-guard
reconciliation belong in the plan; the register owns current disposition.

<!-- Migrated from UPSTREAM-FIXES.md; historical evidence retains its revision. -->

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
[UPSTREAM-PLAN.md](#lookup-key-not-null-task-separate-session).

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

## U17 provenance snapshot (2026-11-11 migration)

Implemented on independent branch `4.1-lookup-key-not-null`, [PR #23724](https://github.com/juju/juju/pull/23724), head `4e7341392f`; open for approval. Declares `INT PRIMARY KEY NOT NULL`, regenerates scalar IDs, validates payload/recovery boundaries and cross-checks decoder inventory against DDL. Explicit typed deltas and the 65-key scope are retained. No PoC integration is recorded.
