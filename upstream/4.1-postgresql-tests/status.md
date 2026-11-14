# 4.1-postgresql-tests

[Integration plan](integration-plan.md) | [H3 contract](h3-plan.md) |
[Dashboard](../../UPSTREAM-PLAN.md) | [Shared direction](../../AGENTS.md)

## Current state and next actions

Observed 2026-11-14. Owner/session:
`1182cd49-fff3-42a1-b3ef-bf58b866213c`; branch `4.1-postgresql-tests`,
worktree `../4.1-postgresql-tests`. Code HEAD and last verified GitHub head:
`6e6887c24ec77236796ef0ea96c279ac3b86341c`. The worktree contains the
uncommitted strict-generator checkpoint described below.

| Stage | State |
|---|---|
| H1: central backend configuration | Committed/pushed, `6e677c2850`. |
| H2: PostgreSQL driver/schema plumbing | Committed/pushed, `57f02035c7`; focused PostgreSQL 16.15 acceptance passed. |
| H3a: source/PoC audit | Complete; five shared-fix groups block full-schema acceptance. |
| H3b: generation | Named sources committed/pushed in `6e6887c24e`; strict generator/check pipeline and initial typed subset validated locally, uncommitted. |
| H3c: full-schema acceptance | Not implemented. No complete generated PostgreSQL assets or constructors. |
| H2.1 / H4 | Sqlair adoption and shared-suite enablement remain separate gates. |

The user approved continuing independent H3b work while shared-source
blockers remain. The latest checkpoint passes SQLite/real-dqlite race
checks, preservation checks and full pre-check; it does not complete H3.
`govulncheck` is absent and non-blocking; no vulnerability scan is claimed.
H2 and the named-source checkpoint were pushed with approval on 2026-11-14.
No PR is recorded; further publication requires separate approval.

Ancestry remains stock upstream
`b36eefc4b34badcdef972c859e96ccb974c592e2` -> layout
`9da8100e161f59ce5a86485e0725db1b8101154d` -> H1 -> H2 -> named sources.
H1 was restored from the verified fork before H2; H2/H3 imported neither
PoC ancestry nor shared-source fixes.

Next actions:

1. Finish independent H3b dependency ordering, foreign keys/views,
   expressions/triggers, and correct identity/changelog allocation.
   Then add complete assets, PostgreSQL constructors and integrated
   generation/freshness checks under the [H3 contract](h3-plan.md).
   Shared defects remain separate fixes, not PoC overrides.
2. Coordinate [H2.1 sqlair adoption](integration-plan.md#h21-adopt-upstream-sqlair-placeholders)
   with the [placeholder comparison](../../investigations/sqlair-placeholders.md).
   Do not assume `WithDialect(PostgreSQL)` or choose a dependency here.
   H3 can proceed independently; final H4 acceptance cannot.
3. Leave separately owned [CI](../4.1-postgresql-tests-ci/status.md) and
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
  SQLite/dqlite APIs, schema bytes, ordering and supported upgrades.
- `DqliteSuite` still rejects PostgreSQL pending H3/H4. PostgreSQL-only
  low-level suites skip for other backends, but explicit PostgreSQL
  configuration/connection failures fail rather than skip or fall back.
- No production PostgreSQL selection, domain SQL fixes, PoC runtime code,
  CI provisioning or other owners' branch changes belong in H3.
- Complete SQL-owning coverage and the end-to-end job budget remain
  unvalidated: under 30 minutes, preferably under 15, including setup,
  compilation and cleanup. Focused timings do not establish that gate.

## H3a actual-source audit (2026-11-14)

Audited clean code head `57f02035c70aac8435c28341b9175980a9014ff5`
and read-only PoC reference
`f3340c67107829347ee750f7e40e806e563411e9`. Neither code checkout was
modified. No code was ported, no fixes were cherry-picked and no overrides
were approved by this audit. Findings below are tied to those revisions,
not the current upstream branch or GitHub review status.

### Complete assembled inputs

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

The source inventory, per-patch hashes and diagnostic program are retained
in session `1182cd49-fff3-42a1-b3ef-bf58b866213c`'s `files/` directory:
`h3-audit.go`, `h3-audit-final.log` and `h3-audit-before.log`. At audit
completion, the code worktree was clean and all source and Go-built patch
bytes were unchanged. No release DDL assets exist in H2's tracked dqlite trees;
`generate/ddlgen` and its historical release-output behavior remain
unchanged. This is H3a evidence, not generated PostgreSQL acceptance.

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
| FK/view dependency order | Genuine backend difference. Resolve referenced keys/indexes, forward references and table cycles; reject missing targets, duplicate definitions and view cycles. H2 declares no `DEFERRABLE` constraints, so distinguish creation-time forward references from deferred enforcement. Test explicit deferred-reference fixtures; do not make all FKs deferred silently. | H3b dependency validator and H3c actual constraint behavior. |
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

### Reusable generator provenance and required changes

Reference files below are all under `generate/pgschemagen/` at PoC
`f3340c6710`. None had been copied at the H3a checkpoint; subsequent
selective reuse is recorded in the strict-generator checkpoint below.

| Component | Reuse assessment |
|---|---|
| `lexer.go`, `split.go` | Reuse token-preserving lexical/splitting structure and literal/comment/function-body fixtures. Add strict malformed/unsupported-construct rejection; tokenization alone is not syntax validation. |
| `translate.go` | Reuse explicit type mapping, typed Boolean seed conversion, identifier quoting and deterministic shortening ideas. Add storage-map integration, collision validation and H2 timestamp handling. Replace the permissive unknown-statement/function pass-through; do not port `DATETIME -> now()` or its changelog helper as proven semantics. |
| `trigger.go` | Reuse parsed trigger structure, OLD/NEW return selection and predicate-in-function handling for subqueries. Validate event/row syntax and supported body statements. Exercise actual guard/rollback behavior, not only generated strings. |
| `order.go` | Reuse forward-FK relocation and deterministic dependency traversal. Strengthen definition/collision detection, supported-reference parsing and trigger/function dependencies; its maps currently overwrite duplicate table/view definitions. |
| `diff.go` | Reuse exact old-line context matching, but reject ambiguous matches, empty/no-op/stale overrides and unsupported diff input. Its current offset search accepts the first matching location, and its tests explicitly allow an empty override. Any retained override needs provenance/removal conditions. |
| `generate.go` | Redesign source identity extraction around the shared assembler. `readSQLFiles` rejects all `.PATCH.sql`; `patchNames` assumes every trailing patch is a trigger. Neither is acceptable. Add deterministic non-writing freshness checks and generated ownership markers; current `writeOutput` deletes arbitrary unproduced entries in `sql/`. |
| `main.go`, `generate_test.go` | Retain controller/model orchestration and useful focused fixtures, not a second schema registry. Add CLI `-check`, version/post-patch fixtures and deliberate missing/extra/drift cases. Current tests do not establish PostgreSQL storage, concurrency or complete H3c coverage. |

Generation ordering must remain:
existing `go generate ./domain/schema` trigger directives, then one
PostgreSQL generation step. `make rebuild-triggers` already invokes
that package (`Makefile:563-567`). Do not invoke `go generate` recursively
from pgschemagen or repurpose facade `rebuild-schema`. Wire a non-writing
freshness command into `tests/suites/static_analysis/schema.sh`; leave
`make rebuild-ddl` / `generate/ddlgen` behavior unchanged.

## H3b strict-generator checkpoint (2026-11-14)

Uncommitted above `6e6887c24e`, using the pinned PoC as a reviewed reference,
not definitive behavior. No PoC override or shared fix was copied.

### Implementation and reference reuse

- `generate/pgschemagen/` provides `-dqlite`, `-dest` and non-writing
  `-check`. It consumes complete named sources at `version.Current`.
  Shared post-patch inventories distinguish excluded future files from
  unregistered inputs; `core/database/schema.HasArgs` rejects bound
  arguments rather than losing them.
- Adapted the PoC lexer/splitter and exact-diff ideas, adding strict
  malformed-input, nested-comment and dollar-delimiter handling, unique
  context matching and range checks. Unknown SQL and empty/no-op/stale/
  ambiguous overrides reject instead of passing through.
- The typed subset covers scalar columns, primary/unique keys, literal
  seed rows, plain indexes and non-key `ADD COLUMN` post-patches. It uses
  declared columns for Booleans and the existing storage map for binary
  `TEXT`. Quoting/shortening detect relation-name collisions and preserve
  `_pkey`; mixed-case quoted identifiers reject pending lookup support.
- Exact H2 clocks use statement time: milliseconds for supported
  `STRFTIME`, whole seconds for `DATETIME`/`CURRENT_TIMESTAMP`. This has
  fixture evidence only, not PostgreSQL storage/long-transaction acceptance.
  The PoC's transaction-time `now()` rewrite was not copied.
- Both namespaces translate/preflight before writing. Headers preserve
  source position, identity/digest and Juju version. Check mode leaves
  contents, modes, mtimes and directories unchanged. Write mode skips
  identical files, writes atomically and removes only owned stale files;
  unowned files, symlinks, directories and unsafe paths reject.
- Overrides need exact `Source`, nonempty `Reason` and `Remove-when`.
  Unused overrides fail; raw snapshots and handwritten diffs have separate
  ownership. Structural overrides reject pending dependency validation.

### Evidence and limits

Passed from the code worktree, including synthetic generation and
freshness mutations for content, missing/extra files, Go patches,
source-only whitespace, registered patches and overrides:

```sh
JUJU_TEST_DATABASE=sqlite go test -race -p 2 -count=1 -timeout 120s \
    ./generate/pgschemagen ./domain/schema/... ./core/database/schema
JUJU_TEST_DATABASE=dqlite make run-go-tests \
    TEST_PACKAGES='./generate/pgschemagen ./domain/schema/... ./core/database/schema' \
    TEST_ARGS='-race -p 2 -count=1 -timeout 120s'
go generate -run triggergen ./domain/schema
git --no-pager diff --exit-code -- domain/schema/dqlite
STATIC_ANALYSIS=1 make pre-check
git --no-pager diff --check
```

Both real CLI modes reject the controller foreign key in
`dqlite/controller/sql/0002-lease.sql` without creating output. The model
fixture rejects `INTEGER PRIMARY KEY` allocation in
`dqlite/model/sql/0001-changelog.sql`. These are generator limitations,
not new shared-source defects.

The audit still matches H3a byte-for-byte, including all patch hashes and
SQLite inventories; trigger regeneration has no drift. Race-compiled
SQLite schema stress completed 12 runs, two processes, zero failures.
Full pre-check passed (414-second static suite, including lint/schema/
`txncheck`); touched Go files use `gci`/`gofmt`. IDE diagnostics are limited;
CLI checks provide substantive compilation evidence.

Logs in the current owner's session `files/`:
`h3b-generator-sqlite-final.log`, `h3b-generator-dqlite.log`,
`h3b-generator-stress.log`, `h3b-audit-after.log`, `h3b-trigger-generation.log`,
`h3b-cli-generate.log`, `h3b-cli-check.log` and
`h3b-pre-check-final.log`. `h3b-generator-files.sha256` fingerprints the
15 validated uncommitted Go files. Temporary stress binaries were removed.

No PostgreSQL acceptance ran here. The incomplete generator is deliberately
not wired into existing generation or static checks; remaining work and
shared blockers are listed above.

## H3b named-source foundation (2026-11-14)

Commit `6e6887c24e` exposes `ControllerPatchSourcesForVersion` and
`ModelPatchSourcesForVersion`; existing DDL constructors use those same
sequences. `PatchSource` preserves full patch values, SQL paths, and
registering-Go-file identities such as `model.go#patch-0038`.
Version-selected post-patches remain last, with the original registries
authoritative. Runtime file-selection behavior is unchanged; strict
inventory rejection belongs to the generator.

Fixtures cover exact assembled sources, single closure evaluation,
bound arguments, non-trigger post-patches, registration order, version
inclusion/exclusion and missing inputs. SQLite/real-dqlite race suites,
both generation consumers, scoped lint/`txncheck` and full pre-check
(402 seconds) passed at this commit. Real-dqlite race stress completed
14 runs with zero failures before the intentional 31-second outer timeout.
The audit matched H3a; trigger regeneration had no drift.
`make rebuild-ddl` produced no release files at prerelease `4.1-beta3`,
so historical release-output generation was not exercised.

Evidence in the current owner's session `files/`:
`h3-sources-sqlite-final.log`, `h3-sources-dqlite.log`,
`h3-trigger-generation.log`, `h3-ddl-generation.log`,
`h3-sources-stress.log`, `h3-sources-lint.log`, `h3-sources-txncheck.log`,
`h3-sources-after-final.log` and `h3-pre-check-no-govulncheck.log`.
`h3-sources-files.sha256` fingerprints its five committed files.

## H2 implementation and provenance (2026-11-14)

Revision `57f02035c70aac8435c28341b9175980a9014ff5`, above H1
`6e677c28505645413b55862e732c28e576ced3b9`. Selectively adapted the PoC
pgx driver with `github.com/jackc/pgx/v5 v5.11.0`, not a verbatim import.

- Explicit connectors default to serializable transactions and UTC.
  Positional `?` and Sqlair parameters are bound as PostgreSQL `$N`;
  native numbered parameters work without mixing representations.
  Quoted literals/identifiers, comments and dollar-quoted bodies are
  preserved. Unquoted `?` is treated as a legacy placeholder, not a
  PostgreSQL JSON operator; this is a known bridge limitation that H2.1
  must resolve before accepting native sqlair PostgreSQL output.
  Invalid empty `IN ()` is deliberately not rewritten or hidden.
- Error classification covers wrapped SQLSTATEs under both build-tag
  variants. Serialization failures and deadlocks retry; connection loss
  and server shutdown do not retry an uncertain commit blindly.
  Existing shared transaction retry machinery is reused.
- PostgreSQL schema application takes a transaction advisory lock at
  READ COMMITTED, applies DDL and persisted version/hash records atomically,
  accepts fresh/identical schemas and rejects incremental upgrades or
  changed/newer hash chains. SQLite bookkeeping SQL and upgrades remain
  unchanged. Error causes survive schema-layer wrapping.
- The test helper reads central configuration, creates a random owned
  schema, bounds admin/scoped pools to 2/4 connections, and closes pools
  before dropping only that schema. Cleanup uses a fresh bounded context,
  not the canceled test context. There is no stale-schema sweep.
- Malformed connection diagnostics do not include the original connection
  string. Neither the driver nor the helper reads PoC environment settings.
  No production backend-selection path was added.

**H3 prerequisite:** PostgreSQL uses SQLSTATE `23505` for primary-key and
unique violations. H2 preserves Juju's distinction using PostgreSQL's
default `_pkey` primary-key constraint suffix. Generated schemas must keep
that convention and not give unique constraints that suffix, or replace
the name-based classification before claiming compatibility.

### Validation

Commands used the shared `_deps` and the approved existing PostgreSQL 16.15
server, supplied privately through central configuration. No PoC environment
reading or new server provisioning was added. Final executable checks
preceded only comment/formatting changes in `57f02035c7`; stress binaries
were rebuilt from that exact commit.

Focused PostgreSQL checks passed with the central selector explicitly set:

```sh
export JUJU_TEST_DATABASE=postgresql
# Supply JUJU_TEST_POSTGRESQL_URL privately.
timeout 150s go test -v -race -p 2 -count=1 -timeout 120s \
    ./internal/database/postgresql/... \
    ./internal/database/drivererrors ./core/database/schema \
    -run 'TestRewriteSuite|TestDriverSuite|TestPostgreSQLSchemaSuite|TestPostgreSQLSuite'
```

Coverage includes backend settings, binding/prepared statements, typed
values/timestamps, constraints, commit/rollback/retry, cancellation,
fresh/identical schema hashes, incompatible-schema rejection and atomic/
concurrent application.

The same unchanged existing package selection passed with race detection
on SQLite and real dqlite. An intentionally invalid unused PostgreSQL URL
was retained for both, demonstrating that other backends ignore it:

```sh
export JUJU_TEST_DATABASE=sqlite
export JUJU_TEST_POSTGRESQL_URL='intentionally invalid unused connection string'
timeout 180s go test -race -p 2 -count=1 -timeout 120s \
    ./internal/database/postgresql/... ./internal/database/drivererrors \
    ./internal/database/testconfig ./internal/database/testing \
    ./internal/database ./core/database/schema ./domain \
    ./domain/schema/... ./domain/sequence/state

export JUJU_TEST_DATABASE=dqlite
timeout 180s make run-go-tests \
    TEST_PACKAGES='./internal/database/postgresql/... ./internal/database/drivererrors ./internal/database/testconfig ./internal/database/testing ./internal/database ./core/database/schema ./domain ./domain/schema/... ./domain/sequence/state' \
    TEST_ARGS='-race -p 2 -count=1 -timeout 120s'
```

- Race-compiled `TestDriverSuite` and `TestPostgreSQLSchemaSuite` each
  passed 20 runs with zero failures under
  `timeout 45s stress -p 1 -timeout 30s -count 20 -failfast`.
- Negative runs verified unreachable-endpoint errors and the retained
  domain-suite PostgreSQL gate, not passing domain coverage.
- Scoped lint/`txncheck` and full `STATIC_ANALYSIS=1 make pre-check`
  passed (7m43s). The optional vulnerability scan was disabled.
- Cancellation readiness observes the blocked advisory lock via
  `pg_locks`, then cancels and joins; it does not depend on hidden
  `pg_stat_activity` fields or arbitrary sleeps.
- Final cleanup left zero `juju-pgtest-*` schemas without sweeping.

Logs: session `e23dc8fe-1c80-49f5-8bb0-059ea237919c`, `files/h2-*-final.log`
and `h2-doc-go.log`. Stress binaries were removed. This is lower-level H2
evidence, not H3/H4 or complete-job acceptance.

## H1 history and evidence (2026-11-12)

The initial configuration patch was tested on `bd970f693c`, then
revalidated above layout `9da8100e16` before H1 commit `6e677c2850`.
Both runs covered `internal/database/{testconfig,testing}`,
`internal/testhelpers` and `domain/sequence/state` with race detection,
SQLite and real dqlite. An invalid unused PostgreSQL URL was retained.
Race-compiled harness stress passed 20 runs per backend with zero failures;
lint, `txncheck` and whitespace checks passed.

Fixtures cover selectors/defaults, compiled-backend availability, URL and
keyword connection forms, retained URLs and environment preservation.
Missing URL, unavailable dqlite and the then-unimplemented PostgreSQL
backend failed with expected diagnostics. These were H1 negative checks;
H2 subsequently added focused PostgreSQL execution.

Evidence: session `0ec926e6-3858-4f43-bc9c-658024127e88`,
`files/testconfig-*.log` and `restacked-harness-*.log`. No PostgreSQL or
CI run occurred at H1. The original 2026-11-11 reservation and subsequent
layout/CI/integration decisions are superseded by the
[integration plan](integration-plan.md); historical PoC timings remain in
[Stage 6](../../stages/06-compatibility.md#database-pool-poc).

## Deferred: PostgreSQL test-suite upstreaming

This is not a stock-upstream breakout or a dispatchable standalone fix.
User decision on 2026-11-10: carry the local changestream shutdown
cancellation fix with or after the future effort to run the upstream test
suite on PostgreSQL, rather than factoring it out now.

PoC commit `7e513ceaa8` in `internal/changestream/stream/stream.go` checks
`errors.Is(err, context.Canceled)` without first applying `errors.Cause`.
Its regression covers direct and wrapped cancellation during normal
shutdown. PostgreSQL produced the real failure; natural SQLite/dqlite
reproduction has not been established. Preserve that distinction when
scoping the future PostgreSQL test-suite work. Finding details and local
validation are in UPSTREAM-FIXES.md under stream shutdown.
