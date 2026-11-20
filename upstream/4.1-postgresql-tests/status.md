# 4.1-postgresql-tests

[Integration plan](integration-plan.md) | [H3 contract](h3-plan.md) |
[Dashboard](../../UPSTREAM-PLAN.md) | [Shared direction](../../AGENTS.md)

## Current state and next actions

Observed 2026-11-20. Branch `4.1-postgresql-tests`, worktree
`../4.1-postgresql-tests`.

- User-authorized rebuild: `b3089f76-ea48-49bf-89e5-dc39370b855e`.
  Previous H3 implementation/evidence owner:
  `1182cd49-fff3-42a1-b3ef-bf58b866213c`; design/planning continuation:
  `5eb9907f-1835-4e64-98a2-d54e0aa2461d`.
- Local code HEAD: `11282b07285ebaeeebac6288a17f3304cb04b63e`; layout parent:
  `9c8f98cd16289cb0e17b9be9ac587d59a683b285`; ordered-source prerequisite:
  `b622931495f0fb3a2242418c82e2fcb152183354`; stock upstream base:
  `59dcd6d951814fa098a11ca3be103314bb6894e8`.
- The worktree is clean. Four harness commits were replayed onto the user's
  rebuilt layout, followed by a fixture-only adaptation. The sole rebase
  conflict retained the source-order documentation in `patch.go` alongside
  the named-source API.
- The obsolete dirty FK-relocation checkpoint is archived, not reapplied.
  The generator requires source-ordered tables and keys and does not relocate
  constraints. See [rebuild evidence](#ordered-layout-rebuild-2026-11-20).
- Nothing was pushed. The last recorded GitHub head is `c59a24d92a`; remote
  state was not refreshed and no PR is recorded.

| Stage | State |
|---|---|
| H1: central backend configuration | Replayed locally as `b45c7d089e`; previously pushed as `985463ee4d`. |
| H2: PostgreSQL driver/schema plumbing | Replayed locally as `90964d6723`; previously pushed as `a80e0b835b`. |
| H3a: source/PoC audit | Complete; shared-source blockers identified below. |
| H3b: generation | Named sources (`accc28f454`) and strict foundation (`b20b4d0715`) replayed locally; ordered-layout fixtures in `11282b0728`. Full translation is incomplete. |
| H3c: full-schema acceptance | Not implemented. No complete generated PostgreSQL assets or constructors. |
| H3.1: empty-collection membership | Planned Juju portability commit in PostgreSQL enablement, not a mandatory separate PR; required before affected H4 acceptance. |
| H2.1 / H4 | Sqlair adoption and shared-suite enablement remain separate gates. |

The branch starts from stock 4.1 plus the explicit source-order cleanup and
layout `9c8f98cd16`; it contains no PoC ancestry or other pending shared-source
fixes. Independent H3b work is approved while source blockers remain.
Focused checks do not establish H3 completion; see
[evidence and limits](#evidence-and-limits). Further publication requires
separate approval.

Next actions:

1. Consume the integrated
   [source-order cleanup](../4.1-schema-dependency-order/status.md) as the
   prerequisite baseline. Reject new unresolved dependencies; do not build
   a generic relocation solver or modify the shared sources here.
2. Continue independent H3b work on identity/changelog allocation, clocks,
   used views/expressions/indexes/triggers, complete assets, PostgreSQL
   constructors and generation/freshness wiring. Translate actual assembled
   constructs, not arbitrary SQLite SQL. Keep shared fixes separate and
   revalidate them at the combined-fix checkpoint.
3. Coordinate [H2.1 sqlair adoption](integration-plan.md#h21-adopt-upstream-sqlair-placeholders)
   with the [placeholder comparison](../../investigations/sqlair-placeholders.md).
   Do not assume `WithDialect(PostgreSQL)` or choose a dependency here.
   [H3.1 empty-collection handling](../sqlair-empty-in/status.md) is a
   separate Juju caller-level portability commit, normally in the
   test-enablement PR. Neither blocks H3; both gate affected H4 acceptance.
4. Complete H3c acceptance before shared-suite enablement. Leave
   separately owned [CI](../4.1-postgresql-tests-ci/status.md) and
   [combined-fix integration](../4.1-postgresql-tests-integration/status.md)
   reservations unchanged. Their records own implementation status.

## Scope and stable contracts

- Central configuration uses `JUJU_TEST_DATABASE=sqlite|dqlite|postgresql`
  and `JUJU_TEST_POSTGRESQL_URL`; domain suites inherit it without reading
  environment variables. Unset selection retains build-tag defaults.
  Incompatible backends fail; SQLite/dqlite ignore even a malformed
  PostgreSQL URL. Backend identity remains distinct from SQL dialect.
- Initial PostgreSQL acceptance targets version 16, fresh creation and
  identical version/hash reapplication. Incremental upgrades are a later
  capability; incompatible schemas must fail. Preserve existing
  SQLite/dqlite APIs, schema bytes, ordering and supported upgrades relative
  to the integrated source baseline. The separate source-order task owns
  its deliberate patch-order/hash-chain changes.
- `DqliteSuite` still rejects PostgreSQL pending H3/H4. PostgreSQL-only
  low-level suites skip for other backends, but explicit PostgreSQL
  configuration/connection failures fail rather than skip or fall back.
- No production PostgreSQL selection, domain SQL fixes, PoC runtime code,
  CI provisioning or other owners' branch changes belong in H3.
- Complete SQL-owning coverage and the end-to-end job budget remain
  unvalidated: under 30 minutes, preferably under 15, including setup,
  compilation and cleanup. Focused timings do not establish that gate.

## Source order, not automatic relocation

The user selected an independent upstream cleanup on 2026-11-18, followed
by a strict source-ordered generator. The
[source-order task](../4.1-schema-dependency-order/status.md) owns the
implementation, release-policy gate and acceptance.

A read-only SQLite audit found no multi-object cycles among tables/FKs
or view-read dependencies; `user` has a valid self-reference. There are
four controller and 34 model forward table-to-target FK edges, including
two and 27 cross-file edges. Model file grouping creates patch-level
cycles, so some definitions must move between files rather than merely
renaming whole patches. The audit did not establish target-key/index
timing, write/trigger side-effect ordering or PostgreSQL acceptance.

The PoC instead relocated forward/cyclic FKs and forward views in generated
PostgreSQL output. That machinery is unnecessary for the observed acyclic
object graph once source order is corrected. Validate dependencies and
reject unresolved references; do not silently move constraints past writes.
Runtime transactional FK deferral remains a separate compatibility issue.

## H3a actual-source audit (2026-11-14)

Baseline: H2 `57f02035c70aac8435c28341b9175980a9014ff5`; read-only
PoC reference `f3340c67107829347ee750f7e40e806e563411e9`. The assembled
inventory was unchanged at the current base. These are source/design
findings, not live upstream review status or complete PostgreSQL acceptance.

### Historical complete assembled inputs

This inventory and its digests describe the pre-ordering H2 baseline.
The rebuilt branch uses `9c8f98cd16` as its preservation baseline; source
bytes and generated triggers match that parent exactly. Do not use the
historical filenames, patch positions or ordered digests as current values.

The inventory calls the actual `ControllerDDL()` and `ModelDDL()` at
`4.1-beta3`, rather than treating SQL files as the complete schemas. Every
leading SQL patch was compared byte-for-byte with its sorted source file.
Both assembled sequences applied successfully to fresh in-memory SQLite
databases. Object counts exclude SQLite internal objects and H2's schema
bookkeeping table; this diagnostic executes patch statements directly.

| Input | Controller | Model |
|---|---:|---:|
| Authoritative base SQL files | 31 (`0000`-`0030`) | 38 (`0000`-`0037`) |
| Generated changestream patches | 29 | 49 |
| Other Go-built patches | 1 | 69 |
| Registered/on-disk post-patches | 0 / 0 | 0 / 0 |
| Complete assembled patches | 61 | 156 |
| Created tables / views | 94 / 20 | 250 / 59 |
| Created triggers / explicit indexes | 89 / 89 | 291 / 192 |

Zero-based patch positions and ownership:

- Controller `0`-`30`: `dqlite/controller/sql/*.sql`; `31`-`59`:
  generated changestream closures registered in `controller.go`;
  `60`: the origin-filtered `secret_backend` immutable-table patch.
- Model `0`-`37`: `dqlite/model/sql/*.sql`; `38`-`86`: generated
  changestream closures; `87`-`131`: generic guards, immutability and
  lifecycle patches from `model.go`/`triggers.go`; `132`-`135`: inline
  machine/start-time, agent-version, offer-endpoint and operation-task
  patches; `136`-`155`: `customModelTriggers()` in `modeltriggers.go`.
- The existing constructors append version-selected registered
  `.PATCH.sql` inputs after all those Go patches. Preserve that order and
  the registry as the single source of truth. Future post-patches are not
  necessarily triggers. Fixtures must cover included/excluded versions,
  declared registration order, missing registrations and missing files.
- Namespace IDs come from `controller.go`/`model.go`, including the
  `10000` generic namespace offset in `triggers.go`. Consume compiled
  patches rather than copying these constants or trigger lists.

The diagnostic's ordered digests are SHA-256 over each patch's unsigned
64-bit big-endian byte length followed by its exact bytes. They are
preservation baselines, **not** Juju's persisted schema hash chain:

```text
controller ec1a871b89759916171bb623aed20b32f78eacbcb9e11aecc28de482fa593f0d
model      74da449c07021e636dca9717090121ab0bceeff873e92dc25c039d39c4d98c05
```

Source reordering deliberately changes these ordered digests and patch
hashes; establish a new integrated baseline after that task lands.

The source inventory, per-patch hashes and diagnostic program are retained
in session `1182cd49-fff3-42a1-b3ef-bf58b866213c`'s `files/` directory:
`h3-audit.go` and `h3-audit-final.log`. Source and Go-built patch bytes were
preserved through the current checkpoint. No release DDL assets exist in
H2's tracked dqlite trees; prerelease `make rebuild-ddl` produced no release
files, so historical release-output generation was not exercised.

### PoC override inventory: seven files, eleven hunks

Counts were refreshed from the pinned PoC `.diff` files, excluding their
generated `*-auto.sql` snapshots. All seven files still target constructs
present in H2. Paths in the first column are relative to the PoC's
`domain/schema/postgresql/`; H2 source paths are under
`domain/schema/dqlite/`.

| Override | Hunks | H2 source and concern | Classification / H3 gate |
|---|---:|---|---|
| `controller/overrides/0007-model.diff` | 1 | `controller/sql/0007-model.sql:98`, predicate-free `LEFT JOIN controller AS ctrli` | Shared source portability fix; preserve models when the singleton controller row is absent. Blocks complete controller generation/application. |
| `controller/overrides/0019-model-last-login.diff` | 1 | `controller/sql/0019-model-last-login.sql:20-25`, bare `time` beside `MAX(time)` | Shared aggregate formulation fix. A PostgreSQL-only `MAX(time)` hides the SQLite timestamp-scan requirement. Preserve latest login, one row per user, ties and `time.Time` scanning. Blocks complete controller generation/application. |
| `model/overrides/0012-secret.diff` | 2 | `model/sql/0012-secret.sql:354,364`, predicate-free model joins in permission/owner views | Shared source portability fixes. Preserve singleton Cartesian semantics and empty-model behavior. Blocks complete model generation/application. |
| `model/overrides/0014-constraint.diff` | 4 | `model/sql/0014-constraint.sql:19,49,58,75`, `allocate_public_ip INT` and three implicit `rowid` replacements | One shared nullable-Boolean type/driver issue plus three shared ordering changes, not four permanent PostgreSQL exceptions. The typed schema and export/import contract need correction; explicit input positions must survive all writers and round trips. Blocks faithful complete model acceptance. |
| `model/overrides/0018-machine.diff` | 1 | `model/sql/0018-machine.sql:144`, predicate-free `JOIN agent_version` | Shared source portability fix; retain omission before target-version initialization. Blocks complete model generation/application. |
| `model/overrides/0020-unit.diff` | 1 | `model/sql/0020-unit.sql:95`, same target-version join | Same shared source portability fix and absence contract for units. Blocks complete model generation/application. |
| `model/overrides/0034-cross-model-relation.diff` | 1 | `model/sql/0034-cross-model-relation.sql:124,127`, duplicate `fk_offer_connection_uuid` on one table | Shared schema-name portability fix; preserve both distinct foreign keys. Blocks complete model application. |

These are shared-fix dependencies, not new assignments or authorization to
modify their source. Existing candidates are recorded in
[the upstream dashboard](../../UPSTREAM-PLAN.md#candidates-needing-bounded-scope-before-dispatch).
The nullable-Boolean correction also depends on the existing
[dqlite branch record](../4.1-dqlite-nullable-boolean/status.md).
`GetApplicationConstraints` explicitly documents insertion order and uses
`ORDER BY tag_order, space_order, zone_order`
(`domain/application/state/application.go:2777-2797`); merely adding
PostgreSQL identities would not establish a shared storage contract.

The base also predates the separate nullable-primary-key fixes for
[lookup keys](../4.1-lookup-key-not-null/status.md),
[charm keys](../4.1-nullable-primary-keys/status.md) and
[exposure scope](../4.1-application-exposure-scope/status.md). For example,
SQLite permits NULL in `lease_type.id INT PRIMARY KEY` and its exported
representation is nullable; PostgreSQL rejects it. A generator-only
nullability change would hide the shared schema/payload defect. Reconcile
and revalidate the actual integrated fixes rather than duplicating them.

### Other translation and compatibility concerns

| Concern / source | Classification and required treatment | Blocker |
|---|---|---|
| Declared `INT`, `INTEGER`, `TEXT`, `BLOB`, `BOOLEAN`, `DATETIME`, `TIMESTAMP` in the actual table inventory | Mechanical mapping, with typed seed/default handling. `INTEGER PRIMARY KEY` needs identity treatment; `INT PRIMARY KEY` must not become an identity accidentally. Preserve nullability and explicit-ID behavior. | Generator implementation/testing, not a shared-fix dependency. |
| `domain/schema/storage.go` | Existing compatibility map identifies 16 byte-bound columns declared `TEXT` (2 controller, 14 model). The PoC type-only translator does not consume it; the PoC separately changed several source declarations. Reuse this schema-owned storage knowledge rather than reintroducing another table/column list or blindly mapping these values to PostgreSQL `TEXT`. | H3b representation work; affected schema/storage checks must prove byte preservation. |
| Seven `STRFTIME` defaults | Controller/model changelog `created_at` and witness `updated_at` (four), `charm.create_time`, `machine_requires_reboot.created_at`, `removal.scheduled_for`. Genuine clock/precision compatibility, not permission to import D56's default/charm-order redesign. | H3b timestamp implementation and H3c storage assertions. |
| `DATETIME('now'[, 'utc'])` in generated/custom triggers | SQLite evaluates current time during statement execution, not once at transaction start; output is whole-second UTC text. PoC `now()` is transaction time. Preserve execution-time behavior deliberately; statement/clock timestamp choice and precision must be explicit, including trigger side effects and long transactions. | PoC rewrite cannot be accepted unchanged. |
| `controller/sql/0007-model.sql:92,144,176` | Three `IIF` calls; `CASE` expresses the same Boolean result portably. No `IFNULL` calls occur in H2's assembled DDL/trigger sources. Existing shared IIF cleanup remains separately owned; a transitional lexical translation must not be described as a root-cause fix. | Does not block independent generator mechanics. |
| `controller/sql/0016-flag.sql:3` | One numeric `BOOLEAN DEFAULT 0`; shared cleanup candidate is `FALSE`. No assembled seed rows populate a table containing a Boolean column at H2. Still test numeric Boolean seed conversion with fixtures, using declared table/column position and explicit insert column lists. | No global substitution or inferred new data contract. |
| Unquoted `user` references | Existing shared quoting candidate. PostgreSQL quoting is a lexical compatibility rewrite only: preserve strings/comments and all references, not just `CREATE TABLE`. | Does not block independent generator mechanics. |
| FK/view dependency order | Consume the separate cleanup's explicit source order. Validate referenced tables/keys/indexes and reject missing targets or duplicate definitions; do not add automatic relocation. No assembled DDL declares `DEFERRABLE`, but model import and recovery use `PRAGMA defer_foreign_keys` (`domain/modelimport/state/model/import.go`, `domain/recovery/state/load.go`). Creation-time ordering does not provide that runtime behavior; coordinate the schema/caller design without silently deferring every FK. | H3b dependency validation and H3c actual constraint behavior; affected H4 import/recovery coverage needs runtime parity. |
| Long names and `_pkey` | Mechanical dialect naming with semantic error-classification consequences. Preserve H2's default primary-key suffix; reject unique names using it. Detect collisions across PostgreSQL relation/function/trigger namespaces after shortening, rather than assuming a short hash makes collisions impossible. | H3b validation and H3c actual error classification. |
| Generic/custom guards, `RAISE(FAIL/ABORT)`, trigger `WHEN` | Genuine compatibility boundary. H2 maps PostgreSQL `P0001` to trigger errors, but SQLite FAIL and PostgreSQL exceptions have different statement/transaction rollback behavior. Prove rejected mutations, no surviving unintended changelog/namespace writes and successful allowed changes; retain old/new-row and NULL predicates. | H3c behavior acceptance; textual rewrite is insufficient. |
| Changelog `id` and `internal/changestream/stream/stream.go:477-489` | The consumer reads only IDs above its watermark. PoC's per-schema advisory-lock trigger allocates after locking because PostgreSQL defaults run before BEFORE triggers. Preserve insert visibility, rollback gaps and explicit-ID semantics; a bare identity sequence can miss a late commit. | H3c deterministic two-transaction, cancellation, independent-schema and bounded stress coverage; no acceptance from string assertions. |

The timestamp translation must preserve at least milliseconds for the
seven defaults and whole seconds for existing explicit `DATETIME` trigger
writes, in UTC through driver reads. In particular, charms inserted by
separate statements in one transaction must not all acquire transaction
start time as an accidental `now()` substitution. Existing ambiguous
same-instant charm ordering is separately registered; H3 does not choose
a new ordering key or modify domain SQL.

## Implemented foundation

### PostgreSQL driver and schema plumbing

H2 selectively adapted the PoC pgx driver with
`github.com/jackc/pgx/v5 v5.11.0`; it did not import PoC runtime selection.

- Connectors default to serializable transactions and UTC. The temporary
  bridge binds raw `?` and Sqlair parameters as `$N`, preserving quoted
  text, comments and dollar-quoted bodies. Native numbered parameters also
  work, but representations cannot be mixed.
- The bridge treats every unquoted `?` as a placeholder, so it would corrupt
  a PostgreSQL JSON operator. H2.1 must remove Sqlair rebinding and
  resolve the raw-SQL boundary before native sqlair acceptance. Invalid
  empty `IN ()` is deliberately not rewritten; H3.1 fixes callers instead.
- Wrapped SQLSTATE classification reuses Juju's transaction retry machinery.
  Serialization failures and deadlocks retry; connection loss and shutdown
  do not blindly retry an uncertain commit.
- Schema application takes a transaction advisory lock at READ COMMITTED
  and applies DDL plus persisted version/hash records atomically. Fresh
  and identical schemas succeed; changed/newer chains and incremental
  upgrades fail. SQLite bookkeeping/upgrades and error causes are preserved.
- The test helper creates a random owned schema, bounds admin/scoped pools
  to 2/4 connections, closes pools before dropping that schema, and cleans
  up with a fresh bounded context. It never sweeps stale schemas. Connection
  diagnostics do not expose the original URL; PoC settings are not read.

PostgreSQL reports both primary-key and unique violations as SQLSTATE
`23505`. H2 distinguishes them using the default `_pkey` suffix; generated
schemas must preserve it and reserve it from unique constraints unless
that classification is deliberately replaced.

### Named sources and strict generation

`ControllerPatchSourcesForVersion` and `ModelPatchSourcesForVersion`
expose the same sequences used by existing DDL constructors. `PatchSource`
retains full patch values, SQL paths and registering-Go-file identities,
such as `model.go#patch-0038`. This avoids the PoC's second file inventory
and its assumption that every non-file patch is a trigger.

The generator selectively reused the pinned PoC's lexer/splitter and
exact-diff ideas, not its permissive pass-through, ordering solver or
transaction-time clock substitution.

- `-dqlite`, `-dest` and non-writing `-check` consume named sources at
  `version.Current`. Shared post-patch inventories distinguish excluded
  future files from unregistered inputs. Bound arguments reject rather
  than being discarded.
- Implemented translation covers scalar columns, primary/unique keys,
  literal seed rows, plain indexes, non-key `ADD COLUMN` and table-level
  FKs whose target key already exists. Self-references, composite keys,
  plain unique-index targets and supported update/delete actions are
  covered. Columns/types, names and collisions are checked.
- Boolean values use declared columns; binary `TEXT` uses the existing
  storage map. Exact supported clock expressions use statement time.
  These clock fixtures do not establish real storage/long-transaction
  acceptance. Mixed-case quoted identifiers remain unsupported.
- Unknown/malformed SQL and empty/no-op/stale/ambiguous overrides reject.
  Overrides require exact `Source`, nonempty `Reason` and `Remove-when`;
  unused overrides fail. Structural changes are not currently allowed.
- Both namespaces translate/preflight before output is written. Headers
  retain source position, identity/digest and Juju version. Check mode
  preserves contents, modes, mtimes and directories. Write mode skips
  identical files, replaces files atomically and removes only owned stale
  files; unowned files, symlinks and unsafe paths reject.

**Current generation limit:** controller `0030-changelog.sql` and model
`0010-changelog.sql` reject `INTEGER PRIMARY KEY` allocation. Both changelog
schemas use `AUTOINCREMENT`, as do controller `0150-autocert-cache.sql`
and `0230-user-ssh-keys.sql`. Views, Go-built triggers, check constraints,
partial/expression indexes and used expressions still need implementation.
For example, `0040-controller.sql` and `0170-user.sql` contain non-plain
indexes. Supporting hypothetical SQL forms is not a goal.

The incomplete generator is deliberately not wired into normal generation
or static checks. Once complete, existing `go generate ./domain/schema`
trigger directives must run before one PostgreSQL generation step.
`make rebuild-triggers` already invokes that package. Do not recurse into
`go generate` or repurpose facade `rebuild-schema`; add non-writing
freshness checking to `tests/suites/static_analysis/schema.sh` and preserve
`make rebuild-ddl` / `generate/ddlgen` behavior. The [H3 contract](h3-plan.md)
owns CLI commands and full acceptance.

### Archived FK checkpoint

The former dirty worktree added deterministic same-patch forward-FK relocation.
It installs each constraint as soon as its target table and matching key
exist, including keys supplied by a later plain unique index. Constraints
are nondeferrable. While a key is pending, only supported table/index
creation may intervene; unresolved references fail at patch end.

This cannot handle actual cross-patch dependencies or an unrelated seed
insert before the target table, as occurs in controller `0007-model.sql`.
It also does not implement runtime `PRAGMA defer_foreign_keys` behavior.
The rebuild archived this checkpoint without applying it. Its fixture
success is not a reason to expand or adopt that design.

## Ordered-layout rebuild (2026-11-20)

The user committed the rebuilt layout as `9c8f98cd16` and authorized the
rebase. `git range-diff` shows unchanged H1, H2 and strict-generator patches;
the named-source commit differs only in conflict context because the
source-order comment is retained. Follow-up `11282b0728` updates the actual
lease/changelog filenames and requires the lease output to exist before
checking its foreign keys. Two added fixtures reject forward tables and
later unique-index keys even when they appear in the same patch.

Preserved local refs:

- `backup/4.1-postgresql-tests-before-ordered-layout-20261120`:
  `c59a24d92ab167313f4393e0651f5ca76c1efcc4`.
- `backup/4.1-postgresql-tests-fk-checkpoint-20261120`:
  `439311de5f4547dc41a54bd93b7614f4a0df48fa`, the stash commit containing
  the original five-file dirty checkpoint. The stash was not popped.

The patch is also retained as
`files/postgresql-tests-obsolete-fk-relocation.patch` in the rebuild session.
No CI/integration refs or other owners' worktrees were changed. Their
reservations remain separate; local CI/integration refs were not recreated.

Validation at `11282b0728`:

- SQLite and real dqlite passed race-enabled generator, schema, schema-core,
  agentbinary, driver/error, central configuration and shared database-helper
  suites. Both selections retained an invalid unused PostgreSQL URL.
  Initial generator failures were stale filename expectations; the final
  generator runs passed without weakening the incomplete-generation gate.
- The broad selection was
  `./generate/pgschemagen ./generate/ddlgen ./domain/schema/...`
  `./domain/agentbinary/... ./core/database/...`
  `./internal/database/testconfig ./internal/database/testing`
  `./internal/database/drivererrors ./internal/database/postgresql/...`,
  with `-race -p 2 -count=1 -timeout 120s`. SQLite used `go test`;
  real dqlite used `make run-go-tests`. Final generator/schema reruns
  cover the fixture adaptation.
- Race-compiled domain-schema binaries passed bounded stress for
  `Test(DependencyOrder|PatchSources|ControllerSchema|ModelSchema)Suite`:
  `timeout 91s stress -p 2 -count 10 -timeout 30s`, ten completed runs
  on each backend, zero failures.
- `go generate ./domain/schema` produced no source or trigger drift.
  `git diff 9c8f98cd16 -- domain/schema/dqlite` is empty. Named-source
  tests verify that the compiled constructors expose the same ordered
  patches and exact file bytes.
- Scoped `golangci-lint`, `txncheck`, `gci` and `git diff --check` passed.
  The IDE build succeeded with limited diagnostics; CLI race compilation
  is the meaningful build evidence for these ignored worktree paths.

Logs are in the rebuild session's `files/` directory:
`ordered-layout-sqlite.log`, `ordered-layout-dqlite.log`,
`ordered-layout-{sqlite,dqlite}-generator-final.log`,
`ordered-layout-stress-{sqlite,dqlite}.log`, `ordered-layout-lint.log`,
`ordered-layout-txncheck.log` and `ordered-layout-regeneration.log`.
Temporary stress binaries were removed.

No central PostgreSQL test URL was configured, so actual PostgreSQL
driver/schema execution was not repeated. Full pre-check, complete generated
schemas, H4, CI and combined-fix acceptance remain unvalidated. This rebuild
does not implement the remaining H3 translation work.

## Evidence and limits

These are retained results from the implementation sessions, not new runs
for this document cleanup. Full-schema PostgreSQL acceptance, H4 coverage,
CI and the complete-job timing gate remain unvalidated.

| Checkpoint | Evidence | Limits |
|---|---|---|
| Archived FK checkpoint, above `c59a24d92a` | Generator/schema race suites on SQLite and real dqlite; PostgreSQL existing/forward-FK fixtures; scoped lint, formatting and whitespace checks. Fixtures cover immediate failures, rollback, cycles, composite keys, cascades, `SET NULL` and nondeferrable metadata. | No full pre-check or vulnerability scan. Bounded real-dqlite race stress completed 54 runs without failure; two runs remained active at the outer timeout. Not full-schema acceptance. |
| Historical `c59a24d92a` tree, checked during the previous rebase before its message-only rewrite | Focused H1/H2 and generator/schema suites on SQLite/real dqlite; PostgreSQL driver/schema and existing-FK fixtures; generator lint. Assembled-schema audit matched byte-for-byte; trigger regeneration had no drift. | Full pre-check and stress were not repeated for that rebase. |
| Earlier named-source/strict-generator snapshots | SQLite/real-dqlite race suites; source/version/post-patch and freshness mutation fixtures; full pre-check including lint/schema/txncheck; source-byte and trigger-preservation checks. | Pre-check does not cover subsequent FK changes. Snapshot fingerprints identify the tested files; no complete PostgreSQL schema was generated. |
| H2 baseline `57f02035c7` | PostgreSQL 16.15 driver/schema tests cover binding/preparation, typed values, constraints, transactions/retries, cancellation, hash compatibility and atomic/concurrent application. Driver/schema race stress each completed 20 runs without failures; full pre-check passed. SQLite/real-dqlite regression suites also passed with an invalid unused PostgreSQL URL. | Lower-level plumbing, not native sqlair, domain-suite or complete-job acceptance. Optional vulnerability scan was disabled. |

Cancellation tests establish advisory-lock readiness through `pg_locks`
before canceling and joining, rather than relying on sleeps or restricted
`pg_stat_activity` fields. Negative tests verify explicit connection errors
and the retained domain-suite gate; they are not passing domain coverage.

Retained evidence locations:

- H3 session `1182cd49-fff3-42a1-b3ef-bf58b866213c`, `files/`:
  `h3-forward-fk-*-final.log`, `h3-forward-fk-stress.log`,
  `h3-forward-fk-audit.log`, `h3-rebase-*.log`,
  `h3b-pre-check-final.log`, `h3b-generator-files.sha256`,
  `h3-pre-check-no-govulncheck.log` and `h3-sources-files.sha256`.
- H2 session `e23dc8fe-1c80-49f5-8bb0-059ea237919c`, `files/`:
  `h2-*-final.log`. H1 configuration evidence is in session
  `0ec926e6-3858-4f43-bc9c-658024127e88`, `files/testconfig-*.log` and
  `restacked-harness-*.log`.
- Source-order audit session `5eb9907f-1835-4e64-98a2-d54e0aa2461d`,
  `files/schema-dependencies.go` and `schema-dependencies.log`;
  the separate task record owns its interpretation and follow-up.

## H4 shutdown follow-up

Carry the changestream shutdown cancellation fix with or after PostgreSQL
suite enablement, not as a new independent stock-backend fix. PoC
`7e513ceaa8` changes `internal/changestream/stream/stream.go` to check
`errors.Is(err, context.Canceled)` without first applying `errors.Cause`.
Its regression covers direct and wrapped cancellation during normal
shutdown. PostgreSQL exposed the failure; natural SQLite/dqlite
reproduction has not been established. Retain that distinction when
porting it; finding details remain in `UPSTREAM-FIXES.md` under stream
shutdown.
