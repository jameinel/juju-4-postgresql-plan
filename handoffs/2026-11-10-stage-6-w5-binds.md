# PostgreSQL PoC: Stage 6 worker W5, fixture binds and test SQL (2026-11-10)

## Objective

Classify all 54 failed leaf tests in nine packages recorded by the
2026-11-10 re-sweep at `09=a3818782aa`. Repair the 49 that are
test-side. Report, but do not change, the five that fail in production
queries (items 5 and 9.4).

Most failures are fixtures that bind integers into `BOOLEAN`, `TEXT`,
timestamp or `bytea` columns (X23, X25), which SQLite accepts by type
affinity.

Follow the
[Worker protocol](2026-11-10-stage-6-parallel-repairs.md#worker-protocol)
for workspace, scope, test commands, commits and the report. Its bind
rule (D55) applies to every item: check the column's declared type
against production writers first.

## Starting state

- The integrator gives you the worktree (`../juju-4-pg-w5`), branch
  (`pg-work/w5-binds`), base commit and `-p` budget.
- Evidence: [06-compatibility.md](../stages/06-compatibility.md),
  [DIALECT.md](../DIALECT.md) and the sweep JSONL named in the
  [parallel-repairs handoff](2026-11-10-stage-6-parallel-repairs.md#starting-state).

## Work and boundaries

Upstream homes: boolean binds use `4.1-boolean-and-aggregate-literals`
(U20, D55 audit; UPSTREAM-PLAN.md). Others are `new` unless stated.

1. **`domain/blockdevice/state`, 6 tests (X25).** The insert helper
   binds `inUse` as an integer (`state_test.go:61-73`) into `in_use
   BOOLEAN` (pgx: "unable to encode 1 into binary format for bool").
   Bind a Go `bool`. Tests: `TestGetBlockDevice`, `TestBlockDevicesOne`,
   `TestBlockDevicesMany`, `TestBlockDevicesFilersOnMachine`,
   `TestMachineBlockDevices` and `TestUpdateBlockDevicesUpdatesProvenance`.
2. **`domain/provisioner/state/model`, 5 tests (X25).**
   - Four fail in `runQuery` (`package_test.go:40-44`) inserting
     `storage_volume_attachment.read_only` as an integer (42804):
     `TestGetProvisioningInfoWithProvisionedModelScopedVolumeAttachment`,
     `...WithProvisionedMachineScopedVolumeAttachment`,
     `...ExcludesUnprovisionedModelScopedVolume` and
     `...ExcludesAttachedVolume`.
   - `TestGetProvisioningInfoConstraintsAllFields` binds `1` for
     `constraint` argument 10 (`allocate_public_ip`). The dqlite schema
     keeps this column non-boolean until canonical/dqlite#883 ships
     (UPSTREAM-PLAN.md checklist); a Go `bool` encodes on both.
3. **`domain/storage/state`, 8 tests.**
   - X25: the block-device helper ending at `import_test.go:1184`
     binds `in_use` as an integer (`TestImportVolumesFoundBlockDevice`,
     `TestGetBlockDevicesForMachinesByNetNodeUUIDsMany`), and
     `base_test.go:373-374` writes `persistent` as the literal `1`
     (`TestStorageInstanceInfoVolumeUnitAttach`,
     `TestStorageInstanceInfoVolumeUnitAttachNoDeviceLinks`).
   - X27: `TestStorageInstanceInfoFilesystemUnitAttach`
     (`instance_test.go:171`),
     `TestStorageInstanceInfoFilesysteamVolumeBackedUnitAttached` (:293),
     `TestCreateStorageInstanceWithExistingFilesystem`
     (`storage_test.go:173`) and
     `TestCreateStorageInstanceWithExistingVolumeBackedFilesystem`
     (:374, :394). Follow the protocol's timestamp rule.
4. **`domain/resource/state`, 5 tests.**
   - 22P02, 3 tests: the test lookup at `resource_test.go:3151-3158` uses
     `COALESCE(revision, '') = COALESCE(?, '')` on an integer column.
     Use a comparison valid on both backends (for example
     `revision IS ?`, or a numeric sentinel). Tests: `TestImportResources`,
     `TestImportResourcesOnLocalCharm` and
     `TestImportResourcesUnitResourceNotMatchingApplicationResources`.
     The `createdAt` check at :3164 may then show X27.
   - X20: `TestUnitResourceRejectsMismatchedResourceName` (:1706) matches
     `FOREIGN KEY constraint failed`. Use `IsErrConstraintForeignKey`
     (N20).
   - X17: `TestListResources` (:2058, fails at :2170) gets
     `with-unit` before `no-unit`. Production `ListResources`
     (`resource.go`, query near :418) has no `ORDER BY`. Apply the
     protocol's order rule: check the service and API callers.
5. **`domain/controllerupgrader/state`, 4 tests: report only.**
   `TestGetAllAgentStoreBinariesForStream` and
   `TestGetAllAgentStoreBinariesForStreamEmpty`, in both
   `controller_test.go` and `model_test.go`, fail inside production
   `GetAllAgentStoreBinariesForStream` (`controller.go:46-55` and
   `model.go:~302`): the query selects `$agentStream.stream_id` with no
   type context, so PostgreSQL infers `text` and pgx cannot encode the
   Go `int` (X9). This is a production defect. Confirm it, put the
   proposed fix (for example `CAST(... AS INTEGER)`, or set the stream in
   Go) in the report, and leave the tests failing.
6. **`domain/upgrade/state`, 1 test.**
   `TestSetControllerDoneCompleteUpgradeEmptyCompletedAt`
   (`state_test.go:353`) runs
   `SET node_upgrade_completed_at = '' WHERE ... controller_node_id = 0`
   at :365-369 (42883 `text = integer`; `controller_node_id` is `TEXT`).
   Quote the node id (a U5 follow-up; juju/juju#23685 fixed the other
   sites). The test then writes `''` into `node_upgrade_completed_at`,
   which is `TIMESTAMPTZ` on PostgreSQL, to simulate legacy data. If no portable equivalent exists, propose a
   skip with reason "PostgreSQL timestamp columns cannot hold an empty
   string", and report how production reads the column.
7. **`internal/worker/upgradedatabase/upgradesteps`, 3 tests (X25).**
   The `TestStep0001_PatchModelConfigCloudType_*` tests bind `0` for
   `cloud.skip_tls_verify`
   (`step0001_patchmodelconfigcloudtype_test.go:31-32`). Check the
   other fixture inserts in the file (:39) the same way.
8. **`domain/ssh/state/controller`, 1 test.**
   `TestGetSSHServerHostPublicKeyEmpty` (`state_test.go:91`) inserts
   the SQLite blob literal `x''` into `public_key bytea` (42804). Bind
   `[]byte{}` instead, and keep the test asserting an empty, non-nil
   result if it does now.
9. **`domain/crossmodelrelation/state/model`, 21 tests.**
   1. `updated_at`, 15 tests: `package_test.go:218-219` and
      `remoteapplication_test.go:2163-2164` insert `relation_status`
      with the literal `updated_at = 0` (42804). Bind a fixed
      `time.Time`. Check whether any test or query depends on this
      value, for example as "oldest status". Tests: 13
      `TestGetOfferDetails*` and `TestGetOfferUUIDByRelationUUID` in
      `offer_test.go`, and
      `TestIsRelationWithEndpointIdentifiersSuspendedStatusSuspending`.
   2. `auto_prune`, 4 tests: `createSecretWithMetadata`
      (`import_test.go:1185-1209`) uses the literal `0` for `auto_prune`
      at :1195. Use `FALSE`, or bind `false` as `secrets_test.go:118`
      does. Tests: the three `TestImportRemoteApplicationSecretGrants*`
      and `TestImportRemoteSecretConsumers`.
   3. rowid (X31), 1 test: `TestCreateOfferEndpointsInsertedInNameOrder`
      (`offer_test.go:315`) checks insertion order with
      `ORDER BY rowid` (:354). The approved rewrite: read the endpoints
      back through the production getter that returns them in canonical
      name order (see `TestGetOfferDetailsDeterministicOrder` at :368),
      and assert that order. Keep the "requested in reverse order" setup.
      Physical insertion order is then untested; if you find a consumer
      that relies on it, stop and report.
   4. Order, 1 test, **report only**:
      `TestGetRelationNetworkIngressSortedOrder`
      (`relation_network_test.go:402`) asserts sorted CIDRs, but
      production `GetRelationNetworkIngress` (`relation_network.go:109-119`)
      has no `ORDER BY`. The test name says the order is a contract, so
      this is a production fix (`ORDER BY cidr`). Report it with the
      callers you checked, and whether the sibling `cidr` queries at
      :164 and :329 have the same gap.

## Tests

All packages finish in a few minutes. Run each package fully on both
backends with the protocol's commands. For `upgradesteps`, use its
package path under `internal/worker/upgradedatabase/`.

## Safety and stop conditions

- Do not change production code or schema, including the two
  report-only items.
- If a column's declared type looks wrong (for example a flag declared
  `TEXT`), report it instead of matching the fixture to it.
