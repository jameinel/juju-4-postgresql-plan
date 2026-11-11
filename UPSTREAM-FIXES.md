# Upstream fix register

Concrete 4.1 fixes and shared portability improvements exposed by the PoC.
Their release value does not depend on finishing PostgreSQL support or HA.
Classes are defined in [PLAN.md](PLAN.md#session-protocol). "Fixed" below
means implemented in the PoC, not reviewed, independently extractable or
merged upstream. Record upstream PR/revision and validation when available.
[UPSTREAM-PLAN.md](UPSTREAM-PLAN.md) gives standalone remaining stock-4.1
tasks. [UPSTREAM-DONE.md](UPSTREAM-DONE.md) records implemented upstream
work, decisions and rationale, including work still in review.
This is the coordinator-maintained finding and PoC provenance register.
Branch records own changing upstream execution, review and CI status.
[UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md) contains only the reusable
procedure for bringing landed fixes back into the PoC. Neither is required
setup or QA for stock-upstream implementation agents.

Upstream fixes follow the
[SQL/schema acceptance contract](UPSTREAM-PLAN.md#purpose-and-sqlschema-acceptance)
(D55): correct the shared SQL/schema root cause, not merely its Go binding or
scanning boundary. The final patch remains SQLite/dqlite-only. PostgreSQL
runtime support and PoC workarounds are separate from upstream acceptance;
a source commit's approach is not a required design.

## Keeping the register current

Workers record new findings and evidence in their assigned record and notify
the coordinator, including when no implementation task exists yet. The
coordinator maintains the defect/rationale, affected paths, source revision,
links to upstream branch records, and eventual PoC reconciliation evidence
here. Distinguish found, implemented, reviewed, landed and reintegrated.

When a finding is ready to assign, create a self-contained branch/task
`status.md` with the problem and required behavior, then reserve/link it in
UPSTREAM-PLAN.md. Do not require this register or PoC setup for upstream QA.
Keep execution details and decisions in that branch record. The coordinator
refreshes the short plan/completed summaries and register mappings at handoff,
landing or a requested sweep, not after every branch edit.
The integration procedure should not change merely because a new finding
or source mapping is added here.

## Extraction gate

For each proposed 4.1 change:

- Identify the dqlite defect or portability benefit and any behaviour delta.
  Explain why the actual SQL and stored types resolve it; successful Go
  conversions or passing permissive-backend tests alone are insufficient.
  If the proposed extraction preserves an incorrect schema or contradicts
  the task's constraints, ask the user before implementing.
- Isolate a coherent patch with no PostgreSQL-only/generated dependencies.
  U1/U2 share a PoC commit but are separate commits on the existing
  `4.1-sqlite-integers` review branch. U8/U9 include PG output to omit.
  Verify the extracted patch; do not assume PoC commits are self-contained.
- Carry focused behavior coverage and SQLite plus real-dqlite evidence on
  the target 4.1 revision. Preserve compatibility with legacy import/export
  payloads through validated boundary conversions where needed; current
  generated types should reflect the corrected schema, not freeze the old
  representation. Check recovery and the release's schema-evolution policy.
  PostgreSQL execution is optional for uncertain semantics and must be
  reported separately; it replaces neither supported backend's checks.
- Record the PoC commit, extraction adjustments, validation and upstream
  disposition (not proposed / in review / merged / deferred).

Upstream PRs are listed under
[Upstream review status](#upstream-review-status). The Status column below
records dated PoC provenance, not live upstream review/CI. Prioritise concrete bugs and
schema-correctness fixes for release review; do not wait for all
portability suites to be green.

The [D55 implementation review](UPSTREAM-PLAN.md#d55-implementation-review-2026-11-06)
owns current rework/merge holds. The
[merged assessment](UPSTREAM-DONE.md#d55-merged-implementation-assessment-2026-11-06)
records follow-up advice, including U4's remaining SQLite fixture expressions
and U13's already-open driver follow-up. Neither is fresh runtime validation.

| ID | Class | Location | Defect on dqlite | Status |
|---|---|---|---|---|
| U1 | bug | `v_revision_updater_application_unit` (model schema) | Grouped by `u.uuid`, so it returned one row per unit instead of one unit count per application (D18). | Fixed on PoC Stage 4 (D18). Branch records: [4.1-sqlite-integers](upstream/4.1-sqlite-integers/status.md). See the dated reconciliation records below for validation. |
| U2 | latent | Model and controller schema, eleven lookup FK columns | Declared `TEXT` while referencing integer lookup IDs. SQLite coercion hid the mismatch, and exports carried the IDs as strings. | Fixed on PoC Stage 4 with U1. Branch records: [4.1-sqlite-integers](upstream/4.1-sqlite-integers/status.md). See the dated reconciliation records below for validation. |
| U3 | latent | `internal/database/bootstrap.go` `InsertControllerNodeID` | Bound a `uint64` to the `TEXT` `dqlite_node_id` column. It worked only because the bootstrap node always uses go-dqlite's `BootstrapID` (high bit clear); `database/sql` rejects a `uint64` with the high bit set. | Fixed on PoC Stage 6; preserve full unsigned node IDs as text. Branch records: [4.1-controller-node-id-text](upstream/4.1-controller-node-id-text/status.md). See the dated reconciliation records below for validation. |
| U4 | latent (test) | `domain/lease/state/migration_test.go` | Seeded leases with `date('now')` and discarded the `StdTxn` error, so a failed seed went unnoticed and the tests checked an empty table. | PoC Stage 6 binds timestamps; the merged upstream fix retained SQLite fixture expressions. Branch records: [4.1-lease-migration-tests](upstream/4.1-lease-migration-tests/status.md); [lease-migration-fixture-timestamps](upstream/lease-migration-fixture-timestamps/status.md). See the dated reconciliation records below for validation. |
| U5 | latent (test) | `domain/changestream/state/state_test.go` `insertControllerNodes`; `domain/upgrade/state/state_test.go` | Insert integer `dqlite_node_id` values into the `TEXT` column, relying on SQLite type affinity. | Stage 6 fixture fixes landed upstream; PoC duplicate dropped 2026-11-05. Branch records: [4.1-controller-node-id-text](upstream/4.1-controller-node-id-text/status.md). See the dated reconciliation records below for validation. |
| U6 | latent (test) | `domain/model/state/testing/model.go` (lines 122, 240) | Uses `""` as an empty string literal. SQLite accepts double-quoted strings only as a legacy misfeature; standard SQL reads `""` as an identifier. | Fixed on PoC Stage 6 (49 files). Branch records: [4.1-sql-test-literals](upstream/4.1-sql-test-literals/status.md). See the dated reconciliation records below for validation. |
| U7 | bug | `v_model_config` (model patch 0027) read by `domain/modelconfig/state` `ModelConfig` | If `model_config` also holds `agent-stream`, the view returns two `agent-stream` rows and `ModelConfig` keeps the last. SQLite's `UNION` sorts, so the `agent_version` stream wins only when it sorts after the config value (DIALECT.md X15). | Fixed on PoC Stage 6; preserve authoritative agent-table values. Branch records: [4.1-model-config-precedence](upstream/4.1-model-config-precedence/status.md). See the dated reconciliation records below for validation. |
| U8 | latent | Controller schema: `user_activation_key.activation_key`, the eight `bakery_config` key columns, `macaroon_root_key.root_key` | Declared `TEXT` but bound as raw random bytes (activation key, bakery key pairs, `dbrootkeystore` root keys). SQLite stores them as BLOBs regardless; `domain/schema/storage.go` `binaryTextColumns` and recovery carried the mismatch, and exports typed them as strings. PostgreSQL rejects them (DIALECT.md X11), breaking `add-user`, the bootstrap worker's bakery initialisation and macaroon minting. | Fixed on PoC Stage 6; PostgreSQL generated paths are not part of upstream extraction. Branch records: [4.1-binary-key-columns](upstream/4.1-binary-key-columns/status.md). See the dated reconciliation records below for validation. |
| U9 | latent | Model schema `removal.force` | Declared `INT` while every writer binds a Go bool or `FALSE` and every reader scans a bool. SQLite stores booleans as integers, so the mismatch was hidden, and exports carried the flag as an integer. PostgreSQL rejects a boolean for `bigint` (DIALECT.md X25). | Fixed on PoC Stage 6; retain legacy 0/1 import conversion. Branch records: [4.1-removal-force-boolean](upstream/4.1-removal-force-boolean/status.md). See the dated reconciliation records below for validation. |
| U10 | portability | `domain/blockcommand/state` `GetBlocks` | Ordered by `rowid`, an implicit SQLite column that PostgreSQL lacks (DIALECT.md X31). On dqlite the order was insertion order. | Fixed on PoC Stage 5 (D31); intentional type-order behavior. Branch records: [4.1-blockcommand-order](upstream/4.1-blockcommand-order/status.md). See the dated reconciliation records below for validation. |
| U11 | latent | `domain/crossmodelrelation/state/model/secrets.go` (two queries) | Group by `sruc.secret_id` but select `sr.uuid` and test `sruc.current_revision` in `HAVING`, so with several consuming units SQLite picks an arbitrary unit row (DIALECT.md X16). Also uses `instr` (X30). | Local fix reconciled 2026-11-05; remote D43 cherry-pick needs comparison after landing. Branch records: [4.1-crossmodel-secret-revisions](upstream/4.1-crossmodel-secret-revisions/status.md); [4.1-local-secret-revisions](upstream/4.1-local-secret-revisions/status.md). See the dated reconciliation records below for validation. |
| U12 | latent | Model schema `relation_status_type.id`, `relation_status.relation_status_type_id` | Declared `TEXT` while the seed rows, `EncodeRelationStatus` and every writer use integers and the offer view compares with `1`. SQLite coercion hid it, and exports carried the ids as strings. pgx will not encode a Go int for `TEXT` (DIALECT.md X23). | Fixed on PoC Stage 6 (D30), with legacy text-ID import conversion. Branch records: [4.1-sqlite-integers](upstream/4.1-sqlite-integers/status.md). See the dated reconciliation records below for validation. |
| U13 | bug | `domain/keymanager/state` `ensureUserPublicKey` | Read the new key id with `LastInsertId`. When the `INSERT ... SELECT` inserted nothing because the fingerprint hash algorithm was unknown, SQLite returned the connection's previous rowid, so the model could be linked to an unrelated key. | Fixed on PoC Stage 5 (D36); merged keymanager follow-up is included in the 2026-11-11 rebase. Branch records: [4.1-keymanager-insert-id](upstream/4.1-keymanager-insert-id/status.md); [4.1-lxd-default-bases](upstream/4.1-lxd-default-bases/status.md). See the dated reconciliation records below for validation. |
| U14 | bug | `internal/worker/dbaccessor/tracker.go` | Leaked database pools. `openDatabase` did not close the pool when setting the foreign keys pragma failed; `newTrackedDBWorker` did not close it when model initialisation failed; `ensureDBAliveAndOpenIfRequired` never closed replacement pools it opened during reopen attempts that then failed their ping. Each leak kept a pool open until the process exited. | Stage 8 `b6a7690d80`; reconciled 2026-11-05 with PostgreSQL residual adaptations. Branch records: [4.1-dbaccessor-pool-cleanup](upstream/4.1-dbaccessor-pool-cleanup/status.md). See the dated reconciliation records below for validation. |
| U15 | latent | Model schema tables with a nullable primary key column: `application_exposed_endpoint_cidr` and `_space` (`application_endpoint_uuid`, NULL is the wildcard endpoint), `charm_container_mount.charm_container_key`, `charm_manifest_base` (`os_id`, `track`, `branch`, `architecture_id`). | SQLite allows NULL in a non-integer primary key and treats NULLs as distinct, so wildcard rows insert, but the key does not stop duplicates of them. PostgreSQL makes every primary key column NOT NULL (DIALECT.md X44). **Found in Stage 9 (2026-10-28):** the first PostgreSQL bootstrap failed in `upsertExposedCIDRs` (`domain/application/state/exposed.go`) exposing the controller application with 23502. Only the exposure tables store NULL. | Charm/exposure cherry-picked under D43 on 2026-11-05; compare against eventual landed results. Branch records: [4.1-nullable-primary-keys](upstream/4.1-nullable-primary-keys/status.md); [4.1-application-exposure-scope](upstream/4.1-application-exposure-scope/status.md). See the dated reconciliation records below for validation. |
| U16 | latent | `domain/agentbinary/state/{controller,model}/state.go` `RegisterAgentBinary` | On a primary key conflict it runs a follow-up `SELECT` in the same transaction. SQLite keeps the transaction usable after the failed statement; PostgreSQL does not (DIALECT.md X18, 25P02 "current transaction is aborted"). The caller's `AlreadyExists` / `AgentBinaryImmutable` handling is never reached. **Found in Stage 9 (2026-10-28):** retries after the U15 failure re-register the binary and fail with 25P02. | D43 cherry-picks replaced by merged shared registration state in the 2026-11-11 rebase. Branch records: [4.1-agentbinary-conflict](upstream/4.1-agentbinary-conflict/status.md). See the dated reconciliation records below for validation. |
| U17 | latent | 48 model and 17 controller single-column `id INT PRIMARY KEY` tables at `8369e6e2a9` | SQLite permits NULL keys; exported `*int64` IDs allow malformed payload rows to reach persistence. | Independent upstream implementation; no PoC integration is recorded. Branch records: [4.1-lookup-key-not-null](upstream/4.1-lookup-key-not-null/status.md). See the dated reconciliation records below for validation. |
| U18 | tooling | `Makefile` `install-dependencies`; static-analysis prerequisites | `install-dependencies` covers Go snap and `libsqlite3-dev`, but not `shellcheck`, `gci`, `golangci-lint` or `stress`. Without `shellcheck`, `STATIC_ANALYSIS=1 make pre-check` stops at dependency checks; without SQLite headers it fails schema generation with `sqlite3.h` missing. The run also reports skipped `go-licenses`, `shfmt` and `govulncheck` checks when those tools are absent. | Developer-tooling finding; no implementation recorded. Branch records: [4.1-install-dependencies](upstream/4.1-install-dependencies/status.md). See the dated reconciliation records below for validation. |
| U19 | latent | `domain/secret/state/state.go` (`updateSecret`, `GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets`, `getSecretsRevisionExpiryChanges`); `domain/unitstate/state/commithook.go` `updateCharmSecret` | Bare columns beside `MAX(revision)` (DIALECT.md X16). They rely on SQLite's single-`MAX` rule to read the latest revision's row; with other aggregates or several join rows, the bare columns (policy, expiry, owner) may come from any row. The revision listing also had no `ORDER BY` (X17). **Found in Stage 9 run 3 (2026-11-05):** the controller unit's uniter failed with 42803. | Stage 5 `5e70fcbc84`, `0ed13ff885`, `3a8f0090c2`. Branch records: [4.1-secret-latest-revision](upstream/4.1-secret-latest-revision/status.md). See the dated reconciliation records below for validation. |
| U20 | portability | `domain/network/state` `GetMachineAppBindings` (`subordinate = 0`), `GetUnitAPIAddresses` (`is_caas = 1`); `domain/agentpassword/state/modelstate.go` `MatchesMachinePasswordHashWithNonce` (bare `mci.instance_id` beside `COUNT`) | Correct on dqlite: booleans are integers, and a machine has at most one cloud instance row. PostgreSQL rejects both (X12/42883, X16/42803). **Found 2026-11-05:** network by package validation before bootstrap run 2 (controller API address path); agentpassword by run 2, where every machine agent login failed. | Stage 5 `30e2bca25a`, `ada2a852ee`; Stage 6 test SQL `fa75871904`. Branch records: [4.1-boolean-and-aggregate-literals](upstream/4.1-boolean-and-aggregate-literals/status.md). See the dated reconciliation records below for validation. |
| U21 | latent | `domain/unitstate/state/state.go` `setUnitStateRelation`; `types.go` `unitRelationStateKeyVal` | Binds Go `int` relation IDs into `unit_state_relation.key`, declared `TEXT`. SQLite stores them by affinity, so the schema and Go type disagree unnoticed (DIALECT.md X23). pgx refuses to encode `int` as text, so every uniter relation-state commit fails on PostgreSQL. **Found 2026-11-05** in the `domain/unitstate/state` re-run once U19 no longer masked it. | Stage 5 `1afd3b9434` is a text-binding workaround, not the upstream integer-schema correction. Branch records: [4.1-unitstate-relation-key](upstream/4.1-unitstate-relation-key/status.md). See the dated reconciliation records below for validation. |
| U22 | latent | `domain/secretbackend/state/state.go` (`ListSecretBackends`, `ListSecretBackendsForModel`, `getK8sSecretBackendForModel`); `domain/removal/state/controller/secret.go` `getK8sSecretBackendForModel`; `domain/storage/state/instance.go` `GetStorageInstanceInfo` | Bare columns with `GROUP BY` (DIALECT.md X16); storage can return an arbitrary mount point when the schema-permitted duplicate attachments share a filesystem and node. **Found 2026-11-05** by PostgreSQL sweep 2 (secretbackend, 13 failures) and by reading. | Stage 5 `52a0eea6ee`, `367f754dd6`, `640d3bcabf`; upstream certificate/attachment semantics differ. Branch records: [4.1-x16-grouped-columns](upstream/4.1-x16-grouped-columns/status.md); [4.1-storage-filesystem-attachment-row](upstream/4.1-storage-filesystem-attachment-row/status.md). See the dated reconciliation records below for validation. |
| U23 | portability (PoC bug) | `domain/lease/state/state.go` claim/extend; `internal/worker/leaseexpiry/store.go` | Stock dqlite evaluates `datetime('now', ...)` on the database leader. PoC `8930466847` instead used each controller's clock, allowing skew to expire leases too early or too late; this regression is not a stock-upstream bug. | Fixed in PoC Stage 5 `76aac853dd`: claims and extensions sample database time per transaction attempt, and expiry compares against `CURRENT_TIMESTAMP`. Full affected packages pass on SQLite, real dqlite and PostgreSQL at `09=a3818782aa`. A portable shared SQL change remains an upstream candidate, not proposed; the PoC's PostgreSQL-specific scanner is not an upstream requirement. |
| U24 | latent | `domain/ssh/state/model/state.go` `EnsureMachineVirtualHostKeyByMachineName` and `EnsureUnitVirtualHostKeyByUnitName` | An expected primary-key conflict is followed by a read in the same transaction. SQLite permits this; PostgreSQL aborts the transaction and the read fails with 25P02 (X18). Existing-key and concurrent-creator handling must retain the first persisted host key, not replace it. | Stage 5 `2f55c00439`; compare the independent upstream fix after landing. Branch records: [4.1-ssh-host-key-conflict](upstream/4.1-ssh-host-key-conflict/status.md). See the dated reconciliation records below for validation. |
| U25 | portability | `domain/controllerupgrader/state/{controller,model}.go` `GetAllAgentStoreBinariesForStream` | The queries project an untyped input stream parameter into every result row; pgx resolves it as text and rejects the Go integer (X9). SQLite accepts it. The field is the same caller-supplied value for every row, not stored data. | Stage 5 `274e1a2e02`; compare the independent upstream fix after landing. Branch records: [4.1-controllerupgrader-stream-result](upstream/4.1-controllerupgrader-stream-result/status.md). See the dated reconciliation records below for validation. |

**U13/U16 reconciliation, 2026-11-11:** The landed key-manager follow-up
and shared agent-binary registration implementation are in the rebased
stack. Their earlier PoC implementations are not retained as competing
paths. See the [reconciliation record](#poc-reconciliation-onto-bd970f693c-2026-11-11)
for residual test work and the incomplete validation.

### Unassigned test findings (2026-11-10)

- **Latent (test), X17:** `domain/model/state/controller/state_test.go`
  assumed query order for activated model UUIDs, although the query has
  no `ORDER BY` and the consumer uses a set. Stage 6 `af63bd985a`
  compares complete unordered contents. Ten PostgreSQL repetitions and
  one real-dqlite run passed with `-race`; see 06-X17. Upstream home is
  unassigned.
- **Portability (test), X27:** Stage 6 `306a0ae3cf` adds a UTC,
  microsecond-precision database fixture helper and uses it for the
  current-time fixtures introduced during the compatibility repairs.
  It preserves explicit non-UTC and precision-boundary fixtures.
  Break out `DatabaseNow()` and its tests upstream separately, with
  representative fixture conversions including charm removal. Keep this
  PoC session focused on bringing up the controller; no upstream branch
  or PR has been created.
- **Latent (test), X17:** `domain/resource/state/resource_test.go`
  assumed database order for unit, application and repository resource
  lists. Stage 6 `555506ac67` compares unordered membership while
  retaining complete resource data, multiplicity and unit ownership.
  At `09=982e0c2fb7` both affected tests passed five times on PostgreSQL
  and real dqlite. Upstream home is unassigned.
- **Portability (test), X35:** `internal/changestream/stream/stream_test.go`
  read fixture change-log IDs with `LastInsertId`, unsupported by pgx.
  W1 uses `INSERT ... RETURNING id`, retaining the retry reset and stream
  ordering assertions. Implemented locally on Stage 6 as `0e1515bfa9`
  (worker source `fe0797778e`). The full stream package passes on
  PostgreSQL and real dqlite at `09=33bbe9d88e`; real-dqlite focused
  stress completed 20 runs without failures. The final worker PG run
  had a separate intermittent shutdown failure described below.
  Not proposed upstream.
- **Latent (test), X17:** `domain/unitstate/state/updateunitports_test.go`
  asserts the order of endpoints returned by `getEndpoints`, whose query
  has no `ORDER BY`. Its production caller uses the endpoints as map keys.
  Both populated and unpopulated tests and the unordered query exist at
  upstream base `5259edff43`. Both assertions failed intermittently on
  PostgreSQL; 20 focused SQLite cases passed. The assertions need
  order-independent checks that retain membership and multiplicity.
- **Portability (test), X16:** `TestUpdateCAASUnitSetsFQDN` in
  `domain/application/state/unit_test.go` and
  `TestEnsureApplicationOnMultipleMachines` in
  `domain/removal/state/model/application_test.go` select bare columns
  alongside `COUNT(*)`. Both fixtures exist at upstream base
  `5259edff43`; PostgreSQL rejects them with 42803. Their count and value
  assertions remain required, as does the FQDN test's repeated-update
  check. No dqlite correctness defect has been established for them.

W3's FQDN test repair is integrated locally as `8496478713`; it preserves
cardinality and every row's values on both updates. The removal counterpart
is committed on W2 as `ad8194c265`, awaiting integration. Neither has been
proposed upstream. The
[Stage 6 re-sweep](stages/06-compatibility.md#postgresql-re-sweep) records
the evidence and tracks the test work under 06-order and 06-X16.

Additional W3 test candidates are integrated locally: representable status
fixtures (`8b1189db74`, X27), explicit endpoint NULL ordering
(`ed4ced01c6`, X47), unordered storage directive assertions
(`0668bcdf63`, latent/X17), specific unique-constraint assertions
(`d4047b2c38`, X20), and complete sequence namespace binds
(`92f3c88638`, X9). The storage assertion preserves multiplicity;
consumers reconcile by name rather than relying on query order. Portable
failure triggers (`7464de87ac`, X46) remain PoC helper usage, not a
standalone upstream helper proposal.

W2 confirmed that `domain.StateBase` deliberately applies `CoerceError`,
masking driver error types while preserving messages. Constraint-class
helpers therefore cannot classify those state-returned errors. This is
expected boundary behavior, not a production defect; the repaired tests
retain constraint-specific messages and operation context.

W2's shared test candidates are integrated locally: secret revision
text binds (`6c65f5e1f8`, latent/X23), job force bool binds
(`1792abeabb`, latent/X25), schedule/job timestamp fixtures
(`12bf06e28e`, `53be340e23`, X27), unordered job comparison
(`b787df58e0`, X17), portable timestamp fixture DDL (`1a80a24fb9`,
X13), foreign-key messages (`154caa56d5`, X20), complete sequence
namespace binds (`48f5ab88d5`, X9), and unit cardinality assertions
(`37c96bd542`, X16). All use `Upstream-Home: new`.
The combined model trigger/bool worker commit (`ac85980e0f`) was split
during integration: PoC helper usage (`bb4ec40957`) has home `none`,
while the boolean literal correction (`ea2a3332f2`) belongs to
`4.1-boolean-and-aggregate-literals`. Worker history was not rewritten.
Relation failure triggers (`61bfd2eaee`, X46) remain PoC helper usage.

W4's test candidates are integrated locally: modelmigration availability
zone binds (`0104af5f6d`, latent/X6), status-model precision
(`6292cb32d5`, X27), secretbackend precision (`6c170aa4a3`, X27),
secret-state precision (`53d3fc725a`, X27), macaroon root-key precision
(`4d10e82eda`, X27), representable root-key bounds (`1a53fd6290`, X28),
operation task-log precision (`d24b792afc`, X27), operation row-count
binds (`e3e16386f0`, latent/X6), imported-operation precision
(`3a31d43367`, X27), and order-independent secret expiry watcher
assertions (`03cd9331d0`, latent/X17). All have `Upstream-Home: new`;
the two X6 changes follow U6. Non-UTC input coverage, exact time
comparisons, bounds and watcher multiplicity are preserved.

W5 is integrated locally. Boolean fixture follow-ups belong to
`4.1-boolean-and-aggregate-literals` (U20): block-device usage
(`9281dbe007`), provisioner attachments (`3ea5198180`) and public-IP
allocation (`7981608f56`), imported block devices (`0f3aebfa2f`),
persistent storage volumes (`97061eea35`), cloud TLS
(`85dbf94401`), and secret pruning (`4e8fae0802`).

W5's `Upstream-Home: new` candidates are storage timestamp precision
(`ed0de74a01`, X27), resource foreign-key classification (`76acf3c533`,
X20), NULL-safe numeric revision comparisons (`59cdf35528`, X10/X49),
empty SSH public-key byte binds (`845439fa58`, X48), relation-status
timestamps (`a033cd444e`, X13), precise secret-permission constraint
messages (`dfa44478f3`, X20/X26), and canonical offer endpoint getter
assertions (`d8c9bcbc44`, X31). The getter rewrite tests externally
ordered results, not physical row insertion order. The controller node-ID
literal correction (`24633d1b47`, latent/X10) is a follow-up to
juju/juju#23685; its legacy empty-timestamp test still fails on PostgreSQL,
and the proposed backend-specific skip is not approved or added.

W6's shared test candidates are integrated locally: network check and
unique-constraint assertions (`40ff7cb706`, `4621d353d4`, X20),
textual permission grants (`b1872be5c2`, latent/X23), explicit permission
fixture cross joins (`682bc1efd5`, X29), immutable-model trigger messages
(`5500e4dae4`, X26), relation fixture precision (`1d6c14b08b`, X27),
order-independent unitstate endpoint assertions (`8712a55006`,
latent/X17), and portable prepared-statement fixtures (`e68252e6cb`,
X32). These all have home `new`. The INTEGER projections retaining
nullable-boolean assertions (`3d4afe2592`, latent/X25) follow
juju/juju#23759. On 2026-11-11 the refreshed #23759 commits
`c9a168b5cd`, `c51a0e05a1` and `39b1dc3cd8` were integrated on PoC
Stage 5 as `0997c97b74`, `0b54417c0b` and `6630d3b862`, respectively.
The superseded Stage 6 machine projection commit `635744b351` was dropped
with user approval. The separate PoC-only sentinel guard `a4d73c395f`
skips PostgreSQL, not real dqlite. Focused three-backend evidence and
remaining gaps are in [Stage 6](stages/06-compatibility.md#nullable-boolean-integration-2026-11-11).
Integration in the PoC does not mean the upstream PR has landed.

W6's network failure triggers (`0d4744cad0`, X46/X50), machine integrity
PRAGMA guard (`2032592a1c`, X21), delete-side-effect trigger
(`389ac9326e`, X50) and abort trigger (`6a6140ade9`, X50) are PoC helper
or backend-specific test adaptations, with home `none`. The worker's
unitstate commit `81755046f6` used a portability classification; its
integration copy `8712a55006` corrects that to latent without rewriting
worker history or changing the tested patch.

### Production findings from parallel repairs (2026-11-10)

These are reports unless explicitly approved below; implementation and
upstream breakout remain separate actions.

- **SSH conflict recovery, X18:** W1 unblocked the model SSH fixtures and
  reproduced PostgreSQL 25P02 in
  `TestEnsureMachineVirtualHostKeyReturnsExistingOnConflict` and
  `TestEnsureUnitVirtualHostKeyReturnsExistingOnConflict`.
  `domain/ssh/state/model/state.go:308-316,436-444` catches an expected
  unique violation then reads in the aborted transaction. Both tests
  pass on real dqlite. **Approved 2026-11-10, U24:** use targeted
  conflict-safe insert SQL followed by a read, preserving existing-key
  and concurrent retry semantics. The user also requested a separate
  upstream breakout, queued as `4.1-ssh-host-key-conflict` in
  UPSTREAM-PLAN.md. PoC Stage 5 `2f55c00439` implements and validates
  the fix at the restacked tip; independent upstream [#23807](https://github.com/juju/juju/pull/23807)
  remains under review.
- **Stream shutdown, confirmed on PostgreSQL; upstream deferred:** W1
  observed `TestStreamSuite/TestWatermarkWriteIsIgnored` fail during clean
  shutdown with `reading change: retry stopped`. IDE debugging reproduced
  the same shutdown failure in `TestOneChange` on PostgreSQL:
  `ctx.Err()` was `context canceled`, `errors.Is(err, context.Canceled)`
  was true, but `errors.Is(errors.Cause(err), context.Canceled)` was false.
  PostgreSQL retry handling wrapped the cancellation; applying `Cause`
  discarded the part of the error chain needed to recognize it.
  The approved local fix checks the full chain in
  `internal/changestream/stream/stream.go`; direct and wrapped cancellation
  regressions were added in `stream_test.go` (PoC commit `7e513ceaa8`).
  The wrapped case failed
  before the fix. The affected suite and bounded race stress passed on
  real dqlite and PostgreSQL. No natural SQLite/dqlite reproduction was
  established; the regression injects the wrapped error, and existing
  cancellation/retry checks passed 100 repetitions on each backend.
  **User decision 2026-11-10:** retain the local fix, but do not extract a
  standalone upstream fix now. Carry it with or after the future effort
  to run the test suite on PostgreSQL upstream. Do not claim a demonstrated
  existing SQLite/dqlite shutdown defect.
- **Controllerupgrader stream parameter, X9:** W5's grouped PostgreSQL
  run reproduced all four `TestGetAllAgentStoreBinariesForStream*`
  failures across the controller and model suites.
  `domain/controllerupgrader/state/controller.go:46-55` and
  `model.go:281-303` project an integer parameter without type context;
  pgx infers text. **Approved 2026-11-10, U25:** remove the SQL
  pass-through parameter and populate each result's stream from the
  method argument in Go. These methods return a fully collected slice,
  not pages or a cursor. Retain the model stream check and existing
  controller-store behavior. Independent upstream breakout
  [#23806](https://github.com/juju/juju/pull/23806) is published from
  `4.1-controllerupgrader-stream-result`. PoC Stage 5 `274e1a2e02`
  implements and validates the fix at the restacked tip; reconcile the
  upstream change after landing.
- **CMR ingress order, local decision 2026-11-10:** W5 found
  `domain/crossmodelrelation/state/model/relation_network.go:109-119`
  lacks ordering while the sorted-ingress test asserts a sequence.
  Focused validation now reproduces unsorted CIDRs on PostgreSQL;
  the unchanged sorted-order test passes on real dqlite.
  The user chose guaranteed lexical ordering. The local state query now
  explicitly orders by CIDR, and state/service contracts document the
  ordering. The sorted-order regression now covers mixed IPv4/IPv6 and
  lexical rather than numeric address ordering; service fixtures follow
  the same contract. Sibling egress queries remain unchanged.
  This is a local decision, not proof of the original upstream intent.
  The `4.1-cmr-ingress-order` task in UPSTREAM-PLAN.md requires raising
  that distinction when preparing the upstream fix: ask maintainers
  whether sorted output was strictly intended or merely incidental
  SQLite order, record their response and report back to the user.
  Upstream [#23805](https://github.com/juju/juju/pull/23805), head
  `1f2411ad8c013246780abcc0e1077db179d51197`, asks that question and
  implements explicit lexical ordering independently on 4.1. Affected
  SQLite and real-dqlite race suites, scoped lint and txncheck passed;
  live controller QA and upstream PostgreSQL execution were not run.
  Review and CI remain pending; several CI jobs failed without a confirmed
  cause. Reconcile the PoC after landing.
  At `09=3a590b446f` plus these local edits, all ten ingress getter
  state cases passed with race detection on PostgreSQL and real dqlite;
  all nine mocked service getter cases also passed. Logs are retained
  in session `fbc8c37e-644c-44ad-8442-79c9a7222baf/files/` as
  `cmr-ingress-{pg,dqlite,service}.log`. Stage 5 integration remains
  pending; this is working-tree evidence, not a restacked-tip result.
  The reviewed change is saved on Stage 9 as `0de25187bf`, pending
  redistribution into the owning stack layer.
- **Resource-list name association, contract resolved 2026-11-10:**
  The user confirmed that resource names, not returned indexes, link
  application and repository resources. Charm revisions can have
  different sets of resource names. Preserve the slice-based
  `ApplicationResources` API; changing it to maps is not necessary.
  The local core type comments state the name-based association and
  unspecified ordering (PoC checkpoint `04d04bee87`). Stage 6
  `555506ac67` repairs the assertions, retaining membership,
  multiplicity, resource-name/UUID/revision associations and unit
  ownership while handling nested resource order independently. Do not
  discard duplicates by converting actual results to maps or impose
  `ORDER BY` merely to retain incidental SQLite order. The focused
  PostgreSQL failure is an assertion-order mismatch, not established
  evidence of incorrect resource contents; the earlier complete
  real-dqlite resource group passed.
- **Provisioner nullable bool, resolved by inspection:** W6 identified
  `domain/provisioner/state/model/state.go:605` scanning nullable
  `sva.read_only` into a bool. The affected PostgreSQL machine
  volume-cleanup case now passes after the test-only INTEGER projections;
  both cleanup cases pass on real dqlite too. SQLair explicitly maps NULL
  to the zero value for non-pointer struct fields, so this bool receives
  false. Normal attachment creation can leave the column NULL; the
  neighboring storage provisioning path also defaults it to false.
  No production defect was established and no production change was made.
  The nullable-Boolean PR juju/juju#23759 preserves NULL distinctions in
  other paths; it does not modify this query, which exposes only a bool.

Worker evidence: `~/pg-poc-work/w1/{all-pg-final,all-dqlite-final,watermark-pg-focused-retry}.log`
at worker head `e6585b9fb4`. Integrator reruns at `09=33bbe9d88e`:
`~/pg-poc-work/integrator/w1-{pg,dqlite}.log`. They reproduce only the
two SSH failures; all five packages pass on real dqlite.

### U17: nullable INT lookup primary keys (independent PR in review)

See the [branch record](upstream/4.1-lookup-key-not-null/status.md#u17-nullable-int-lookup-primary-keys-independent-pr-in-review).

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

Branch/finding mappings below link to the execution records. They are not a
second review/CI dashboard. Most PRs target `juju/juju:4.1` from
`jameinel/juju`; the recorded mainline integration targets `main`.
Historical heads and status were moved into branch records with their dates.
Consult those records and GitHub before acting. D55's design assessment
does not establish current CI, QA or PoC reconciliation.

| Branch record | Findings | Recorded PR |
|---|---|---|
| [`4.1-sqlite-integers`](upstream/4.1-sqlite-integers/status.md) | U1, U2, U12 | [#23680](https://github.com/juju/juju/pull/23680) |
| [`4.1-binary-key-columns`](upstream/4.1-binary-key-columns/status.md) | U8 | [#23682](https://github.com/juju/juju/pull/23682) |
| [`4.1-model-config-precedence`](upstream/4.1-model-config-precedence/status.md) | U7 | [#23683](https://github.com/juju/juju/pull/23683) |
| [`4.1-keymanager-insert-id`](upstream/4.1-keymanager-insert-id/status.md) | U13 | [#23684](https://github.com/juju/juju/pull/23684) |
| [`4.1-controller-node-id-text`](upstream/4.1-controller-node-id-text/status.md) | U3, U5 | [#23685](https://github.com/juju/juju/pull/23685) |
| [`4.1-lease-migration-tests`](upstream/4.1-lease-migration-tests/status.md) | U4 | [#23686](https://github.com/juju/juju/pull/23686) |
| [`4.1-crossmodel-secret-revisions`](upstream/4.1-crossmodel-secret-revisions/status.md) | U11 | [#23688](https://github.com/juju/juju/pull/23688) |
| [`4.1-removal-force-boolean`](upstream/4.1-removal-force-boolean/status.md) | U9 | [#23690](https://github.com/juju/juju/pull/23690) |
| [`4.1-changestream-latest-row`](upstream/4.1-changestream-latest-row/status.md) | X16 | [#23691](https://github.com/juju/juju/pull/23691) |
| [`4.1-blockcommand-order`](upstream/4.1-blockcommand-order/status.md) | U10/X31 | [#23692](https://github.com/juju/juju/pull/23692) |
| [`4.1-operation-query-portability`](upstream/4.1-operation-query-portability/status.md) | X34, X33 | [#23693](https://github.com/juju/juju/pull/23693) |
| [`4.1-objectstore-conflicts`](upstream/4.1-objectstore-conflicts/status.md) | X36 | [#23694](https://github.com/juju/juju/pull/23694) |
| [`4.1-sql-standard-syntax`](upstream/4.1-sql-standard-syntax/status.md) | X37, X38 | [#23695](https://github.com/juju/juju/pull/23695) |
| [`4.1-sql-test-literals`](upstream/4.1-sql-test-literals/status.md) | U6 | [#23696](https://github.com/juju/juju/pull/23696) |
| [`4.1-nullable-primary-keys`](upstream/4.1-nullable-primary-keys/status.md) | U15 (charm keys) | [#23710](https://github.com/juju/juju/pull/23710) |
| [`4.1-local-secret-revisions`](upstream/4.1-local-secret-revisions/status.md) | U11 follow-up | [#23720](https://github.com/juju/juju/pull/23720) |
| [`4.1-application-exposure-scope`](upstream/4.1-application-exposure-scope/status.md) | U15 (exposure + combined guard) | [#23723](https://github.com/juju/juju/pull/23723) |
| [`4.1-lookup-key-not-null`](upstream/4.1-lookup-key-not-null/status.md) | U17 | [#23724](https://github.com/juju/juju/pull/23724) |
| [`4.1-dbaccessor-pool-cleanup`](upstream/4.1-dbaccessor-pool-cleanup/status.md) | U14 | [juju/juju#23726](https://github.com/juju/juju/pull/23726) |
| [`4.1-agentbinary-conflict`](upstream/4.1-agentbinary-conflict/status.md) | U16/X18 | [#23728](https://github.com/juju/juju/pull/23728) |
| [`4.1-lease-pin-conflict`](upstream/4.1-lease-pin-conflict/status.md) | X18 | [#23729](https://github.com/juju/juju/pull/23729) |
| [`4.1-dbrepl-latest-row`](upstream/4.1-dbrepl-latest-row/status.md) | X16 | [#23730](https://github.com/juju/juju/pull/23730) |
| [`4.1-sequence-upsert-target`](upstream/4.1-sequence-upsert-target/status.md) | X3/X4 | [#23731](https://github.com/juju/juju/pull/23731) |
| [`4.1-drop-iif`](upstream/4.1-drop-iif/status.md) | X7 | [#23751](https://github.com/juju/juju/pull/23751) |
| [`4.1-cte-membership-subqueries`](upstream/4.1-cte-membership-subqueries/status.md) | X24 | [#23750](https://github.com/juju/juju/pull/23750) |
| [`4.1-pr-description-guidance`](upstream/4.1-pr-description-guidance/status.md) | Agent guidance | [#23755](https://github.com/juju/juju/pull/23755) |
| [`4.1-lxd-default-bases`](upstream/4.1-lxd-default-bases/status.md) | U13 follow-up, test diagnostics, dqlite CI | [#23756](https://github.com/juju/juju/pull/23756) |
| [`4.1-dqlite-nullable-boolean`](upstream/4.1-dqlite-nullable-boolean/status.md) | Nullable Boolean, dqlite CI | [#23759](https://github.com/juju/juju/pull/23759) |
| [`4.1-controller-export-null`](upstream/4.1-controller-export-null/status.md) | Controller backup NULL fidelity | [#23765](https://github.com/juju/juju/pull/23765) |
| [`4.1-install-dependencies`](upstream/4.1-install-dependencies/status.md) | U18 | not proposed |
| [`4.1-secret-latest-revision`](upstream/4.1-secret-latest-revision/status.md) | U19/X16 | not proposed |
| [`4.1-unitstate-relation-key`](upstream/4.1-unitstate-relation-key/status.md) | U21/X23 | [#23769](https://github.com/juju/juju/pull/23769) |
| [`4.1-storage-filesystem-attachment-row`](upstream/4.1-storage-filesystem-attachment-row/status.md) | U22/X16 (latent bug) | [#23766](https://github.com/juju/juju/pull/23766) |
| [`4.1-x16-grouped-columns`](upstream/4.1-x16-grouped-columns/status.md) | U22/X16 | [juju/juju#23784](https://github.com/juju/juju/pull/23784) |
| [`4.1-boolean-and-aggregate-literals`](upstream/4.1-boolean-and-aggregate-literals/status.md) | U20 | not proposed |
| [`4.1-controllerupgrader-stream-result`](upstream/4.1-controllerupgrader-stream-result/status.md) | U25 | [juju/juju#23806](https://github.com/juju/juju/pull/23806) |
| [`4.1-ssh-host-key-conflict`](upstream/4.1-ssh-host-key-conflict/status.md) | U24 | [juju/juju#23807](https://github.com/juju/juju/pull/23807) |
| [`4.1-cmr-ingress-order`](upstream/4.1-cmr-ingress-order/status.md) | X17 | [juju/juju#23805](https://github.com/juju/juju/pull/23805) |
| [`4.1-dbrepl-output`](upstream/4.1-dbrepl-output/status.md) | X16 follow-up | [juju/juju#23757](https://github.com/juju/juju/pull/23757) |
| [`4.1-into-4.2`](upstream/4.1-into-4.2/status.md) | Mainline integration | [juju/juju#23775](https://github.com/juju/juju/pull/23775) |

Notes:

- Active saved PR descriptions are in `upstream/<branch>/pr.md`; replies
  live beside them. Archived drafts remain under `archive/pr-descriptions/`.
- The open PRs in the 2026-11-09 snapshot were mergeable but GitHub required
  review. CI states are revision-specific, not evidence of PoC acceptance or
  a diagnosis of failing jobs; skipped checks are not passes. See
  UPSTREAM-PLAN.md for review, QA, CI and stack actions.
- juju/juju#23730 completes the separate dbrepl latest-row SQL task. No PoC
  integration was performed; dbrepl remains outside the PostgreSQL PoC
  scope. The output-helper refactor merged in #23757 on 2026-11-09 as
  `775513d9df` at head `0583a1972c`; live REPL QA was not recorded.
  Historical old/new SQLite/Dqlite checks and the
  corrected REPL-only scope are preserved in
  [the execution evidence](upstream/4.1-dbrepl-latest-row/status.md#dbrepl-latest-row-selection-x16).
- juju/juju#23720 and #23726 merged on 2026-11-04 and were reconciled into
  the PoC on 2026-11-05 (default backend only; see
  [the reconciliation record](#poc-reconciliation-onto-5259edff43-2026-11-05)).
  U14's independent upstream implementation, nine regressions and package
  race/stress results are preserved in
  [the pool-cleanup evidence](upstream/4.1-dbaccessor-pool-cleanup/status.md#dbaccessor-pool-cleanup-u14).
- juju/juju#23685 also fixes the upgrade test half of U5.
- juju/juju#23696 also fixes double-quoted string literals in production
  SQL, not only in tests.
- X7 is in #23751; X24 is in #23750.
  Continue those implementations through the tracking links in UPSTREAM-PLAN.md.
  X18 is represented by the
  merged U16 agent-binary fix and the separate `PinLease` review above.
- U16's merged shared-state implementation replaced the earlier PoC
  cherry-picks in the 2026-11-11 rebase; validation remains incomplete.
  `PinLease` and
  sequence allocation already have independent review branches; continue
  those rather than starting new implementations.
  Developer tooling and the five-query latest-secret rewrite remain new
  implementation tasks in UPSTREAM-PLAN.md.
- U15 was written upstream first, with no PoC source commit. The published
  split keeps charm keys in juju/juju#23710 and exposure scope in stacked
  juju/juju#23723. Both are needed before the next PostgreSQL bootstrap;
  charm alone leaves the wildcard failure intact. Both were cherry-picked
  onto Stage 5 under D43 on 2026-11-05 (commits in the U15 entry).
  Validation/publication details below are historical and revision-specific;
  they do not establish coverage of the current branch heads.

### Nullable primary-key base refresh (2026-10-30)

See the [branch record](upstream/4.1-nullable-primary-keys/status.md#nullable-primary-key-base-refresh-2026-10-30).

### Nullable primary-key split validation (2026-10-30, pre-refresh)

See the [branch record](upstream/4.1-nullable-primary-keys/status.md#nullable-primary-key-split-validation-2026-10-30-pre-refresh).

### Integer-schema review follow-up (2026-10-29)

See the [branch record](upstream/4.1-sqlite-integers/status.md#integer-schema-review-follow-up-2026-10-29).

### Separate agent-workflow guidance

See the [branch record](upstream/4.1-pr-description-guidance/status.md#separate-agent-workflow-guidance).

## PoC reconciliation onto bd970f693c (2026-11-11)

The local stack was rebased from `5259edff43` to upstream `bd970f693c`.
Recovery tags `pg-poc-before-upstream-bd970f693c/00` through `/09`
retain the original tips. U16's two old cherry-picks were dropped in
favor of merged #23728, including its shared registration refactor.
The merged #23756 key-manager fix replaces the earlier insertion logic;
its PostgreSQL-specific test reconciliation is recorded in
[Stage 6](stages/06-compatibility.md#upstream-rebase-2026-11-11).
Upstream #23592 adds a charm removal type; `04=c0f57b0134` regenerates
the PostgreSQL model DDL. The Stage 8 bootstrap reconciliation retains
upstream's dqlite bind-address resolver behind the PoC database
initialiser. Stage tips and partial checks are in PLAN.md and Stage 6.

This rebase is **not fully validated or published**. The PostgreSQL
charm-removal timestamp fixture is repaired and its focused suite passes
on all three backends. The final-tip sweeps completed with failures; other
rebase checks remain outstanding.

## PoC reconciliation onto 5259edff43 (2026-11-05)

Inputs: old base `a8591fe1b8`, old tips tagged
`pg-poc-base-a8591fe1b8/<stage>` (`00=5b1d9dac4e` … `08=950bba743e`),
new base upstream 4.1 `5259edff43` (merge of juju/juju#23726). Method:
one interactive `git rebase --update-refs` from the bottom, dropping
landed duplicates; then D43 cherry-picks onto Stage 5 and
`rebase --update-refs` of 06–08.

Dropped as landed (PoC commit → upstream):

| Finding | Dropped PoC commit(s) | Upstream | Residual kept |
|---|---|---|---|
| U1/U2 | `32183700c2` | #23680 | none |
| U12 | (part of #23680) | #23680 | PostgreSQL offer relation-status override removed: `3ebc274c90` |
| U3 | `988f8aab04` | #23685 | none |
| U4 | `ad7360b88f` | #23686 | none |
| U5 | `494393013a` (empty after U3) | #23685 | none; upgrade half fixed by #23685 |
| U6 | `6d9c5520df` (empty) | #23696 | none |
| U8 | `0c642e9851` | #23682 | none |
| U9 | `97e876f42b` | #23690 | none |
| U10/X31 | `fde2204e15` | #23692 | none |
| U13 | `c45566ec04` | #23684 | none |
| U14 | `17cf0d3dff` | #23726 | PoC backends adopted upstream's `currentDB` and close-replacement structure (see U14) |
| X16 changestream | `330e6cc208` | #23691 | none |
| X36 | `aebbeb6739` | #23694 | none; upstream `ErrMetadataUUIDAlreadyExists` semantics used throughout |
| X37/X38 | `1699be8c09`, `551ba2f9db` | #23695 | none |
| U11 follow-up | `f2b508d9fb` | #23720 | none; `domain/secret/state` matches `edb2aef5b9` |

Kept: U11 `d61fb4a73b` → `2f81195ea3` (#23688 open); U7 → `42e1d20a17`
(#23683 open); lease pin `4d90c750f8` (#23729 open); the Stage 5
equivalents of #23750 (X24) and the X7 branch, which were not
cherry-picked.

Conflicts resolved by hand: the leaf dialect package (`bcf20cf7d3`, was
`3790bb37f1`; `strftime('%s')` hunk) and the tracker backends
(`f77a1ffb9d`, was `597d915c1b`; see U14).

Generated artifacts: `4248ede57c` regenerated the PostgreSQL DDL at
Stage 4 for the new base (18 files); `52b6ba047c` regenerated it at
Stage 5 after the U15 cherry-picks. Each of the eight overrides was
removed in turn and regenerated; all still change the output, so none
was dropped. Since then U12 removed `0032-offer.diff` (`abd0d5a0ce`),
leaving seven at `pg-poc/09-bootstrap`. Freshness check: no stale generated file at any stage tip.

Validation: adjacent ancestry at every stage; `go build ./...` and
`go vet` at the top; on the **default backend only**:
`internal/worker/dbaccessor` (`-race`), `domain/schema/...`,
`core/database/schema`, `generate/pgschemagen`, `domain/application/...`,
`domain/agentbinary/...`, `domain/modelimport/...`,
`domain/network/state`, `domain/removal/state`, recovery and export.
PostgreSQL validation is outstanding: no server on this machine.

All nine stage tips recorded in PLAN.md are published to `jameinel/juju`,
verified on 2026-11-05; the top is `pg-poc/08-dbaccessor=98847cacb4`.
Publication does not satisfy the outstanding PostgreSQL gate.

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
| U15 | Upstream only. Original combined `43eed70758` is historical and backed up. Initial split: charm `4df04695d8`, exposure `3c90a7ded3` on `9ea82c03bd`. Published refreshed split on `8369e6e2a9`: charm `7001a1b74b` + `ba21f48ad3` in juju/juju#23710, exposure `4d9518117c` + `f31de64e52` in juju/juju#23723. PoC D43 cherry-picks 2026-11-05 of the current heads: see the U15 register entry. |
| U16 | Upstream only (juju/juju#23728 `323807680f` + `6a66e5ed1b`); PoC D43 cherry-picks `fc66010f1a` / `c34b912abb`. |
| X16 change-stream query | `48f0c5e1a9` |
| U10/X31 | `a8301140de` |
| X34/X33 | `18886acfdf` / `e2258798fe` |
| X36 | `d658b402a6` |
| X37 / X38 | `0703122e2a` / `18d679a2b7` |
| U6/X6 | `8a60208176` |
| X3/X4, X24, model IIF | `ff382936b3`, `fda7add768`, `9499d95ca6` |
| X18 | `01316d98fe`, `bccb8d37b4` |

Source-specific cautions:

- `f474710187` (now `8930466847`) mixes syntax, aggregation and lease timing changes. U4's
  `formatTimestamp` depends on it; that does not make the broad rewrite
  an upstream prerequisite. Use upstream-native helpers for the task.
- U1/U2 share a PoC commit but were split in their review branch; do not
  duplicate the unit-count fix while adding later integer corrections.
- U3 and the X16 latest-row source patches lacked dedicated regressions.
  Existing code is evidence, not proof of upstream acceptance.
- Cumulative generated files can contain other findings' changes. Inspect
  the assigned source rather than treating its entire tree as the fix.
