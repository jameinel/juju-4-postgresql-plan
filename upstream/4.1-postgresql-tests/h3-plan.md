# H3: generated PostgreSQL controller and model schemas

[Branch record](status.md) | [Staged integration plan](integration-plan.md) |
[Shared direction](../../AGENTS.md)

Prepared 2026-11-14. This is the implementation contract, not completion
evidence or authorization to publish. The [branch record](status.md) owns
current progress: H3a is complete, H3b is partial and H3c is not implemented.
Do not silently expand into H4 or a shared fix.

Decision update, 2026-11-18: the user selected an independent
[upstream source-order cleanup](../4.1-schema-dependency-order/status.md)
instead of extending automatic PostgreSQL dependency relocation. The actual
table/view graph has no multi-object cycles; the current model file grouping
does. Consume the explicit source order after that cleanup integrates, keep
the generator strict, and reject new unresolved dependencies. Do not modify
the shared SQL sources on this harness branch or expand a general solver.
Preservation checks below apply relative to the integrated source baseline;
the separate task owns its deliberate patch-order/hash-chain changes.

## Starting point and boundaries

Current starting point, 2026-11-20: user-authorized local rebuild at
`11282b07285ebaeeebac6288a17f3304cb04b63e`, above layout
`9c8f98cd16289cb0e17b9be9ac587d59a683b285` and ordered sources
`b622931495f0fb3a2242418c82e2fcb152183354`, on stock base
`59dcd6d951814fa098a11ca3be103314bb6894e8`. The source-order prerequisite
is integrated; preserve this baseline's bytes and ordering. Automatic
FK relocation is archived outside the active branch. The branch record
owns rebuild evidence and the remaining H3 gates.

Historical dispatch baseline, 2026-11-14:

Work in `../4.1-postgresql-tests`, branch `4.1-postgresql-tests`, from
H2 `57f02035c70aac8435c28341b9175980a9014ff5`. That worktree was clean
when prepared. Recheck its head, dirty files and reservation before edits;
do not reset, rebase or recreate another owner's checkout.

The layout parent is `9da8100e161f59ce5a86485e0725db1b8101154d`, above
stock upstream base `b36eefc4b34badcdef972c859e96ccb974c592e2`. Do not
refresh that base or import pending correctness fixes as an incidental
generator change. Refreshing later requires regeneration and revalidation.

Read the source checkout's root architecture/domain rules and
`domain/AGENTS.md`, `domain/schema/AGENTS.md`; read nested guidance before
adding further paths. PostgreSQL acceptance targets version 16, fresh
creation and identical version/hash reapplication. Incremental PostgreSQL
upgrades remain unsupported. Existing SQLite/dqlite APIs, schema bytes,
patch/hash ordering and supported upgrades must remain unchanged.

Allowed implementation surfaces:

- New `generate/pgschemagen/` code and focused fixtures/tests.
- New checked-in `domain/schema/postgresql/{controller,model}/` assets,
  explicit PostgreSQL schema assembly and focused schema tests.
- Minimal shared patch/source extraction in `domain/schema/` if needed
  to avoid a second registry. Preserve existing assembled patch bytes.
- Generation/freshness commands and directly related documentation or
  static-analysis wiring.

Not included: enabling PostgreSQL in `DqliteSuite`, changing domain state
SQL, choosing/updating sqlair, controller runtime selection, production
workers, CI provisioning, branch restacking or combined-fix integration.
Do not modify existing dqlite DDL/triggers to make translation easier.

## H3a: inventory and classify before porting

Use the actual H2 checkout as the source, not generated PoC SQL. Relevant
entry points:

| Input | Contract to preserve |
|---|---|
| `domain/schema/controller.go` | `ControllerDDLForVersion`, generated/generic triggers and registered controller post-patches. |
| `domain/schema/model.go` | `ModelDDLForVersion`, all generated, generic and custom model triggers, then registered post-patches. |
| `domain/schema/{patch,triggers,modeltriggers,storage}.go` | File selection, assembled patch order, namespace IDs, trigger predicates and changes. |
| `domain/schema/dqlite/{controller,model}/sql/` | Authoritative source DDL, seed data, views and registered `.PATCH.sql` inputs. |
| `generate/{triggergen,ddlgen}/`, `Makefile` | Existing generation flow and historical release DDL; do not repurpose facade `rebuild-schema`. |
| `core/database/schema`, `core/database/dialect` | H2's explicit dialect application, atomic bookkeeping and fresh-only guard. |

Both post-patch registries are empty at H2. Test the nonempty contract with
fixtures: generation must consume every applicable registered post-patch
after triggers, and reject missing/unregistered inputs. Never assume every
patch after the SQL-file prefix is a Go trigger or reject all `.PATCH.sql`.

Read-only reference: PoC `f3340c67107829347ee750f7e40e806e563411e9`,
`generate/pgschemagen/{main,generate,lexer,split,translate,trigger,order,diff}.go`
and its tests. Record which pieces are ported or changed; do not cherry-pick
PoC ancestry or modify that worktree.

The reference has useful lexical translation, dependency ordering and
exact-match override mechanics. Its SQL-file enumeration rejects
post-patches, and its patch naming assumes trailing patches are triggers.
Those assumptions are not acceptable in this harness.

Record a per-concern audit in the branch record, with source paths,
classification, needed upstream fix/decision and whether H3b/H3c is
blocked. Audit the integration plan's existing seven-file/eleven-hunk PoC
override inventory; refresh the actual count rather than relying on it.
Distinguish:

- Mechanical dialect translation: types, identifier quoting/length,
  identity syntax, trigger syntax and dependency ordering.
- Genuine semantic compatibility: timestamp evaluation/precision,
  deferred references, immutable/guard trigger behavior, and changelog
  visibility. Specify observable behavior and regression cases.
- Shared defects: predicate-free joins, invalid aggregation, incorrectly
  typed Boolean data, implicit `rowid` ordering and duplicate constraint
  names. Do not bless these as permanent PostgreSQL overrides.

Seven `STRFTIME` defaults remain in the H2 source: controller changelog
(2), model changelog (2), charm, machine and removal scheduling (1 each).
Preserve their intended UTC/time semantics deliberately; the newer PoC
removed that translation after a separate schema redesign. Do not import
that redesign or blindly equate transaction-time and statement-time clocks.

**H3a exit:** complete input/override inventory, reusable-code provenance,
and a bounded blocker list. If correctness requires a shared fix or
unsettled domain behavior, stop that part and ask the coordinator. Do not
copy its patch, assign another owner, weaken coverage or add a silent cast.
Generator-only work may proceed only where independent of that blocker;
H3 is not complete until both full schemas pass.

## H3b: deterministic generation and explicit assembly

1. Port only reviewed generator components. Consume the complete assembled
   schema for the selected Juju version, including Go-built triggers and
   applicable post-patches. Keep source identity/order traceable; do not
   duplicate trigger lists or version registries in the generator.
2. Generate checked-in controller/model SQL in the explicit dependency-valid
   source order established by the separate cleanup. Validate tables,
   foreign keys, views and trigger functions without automatic relocation.
   Reject unsupported syntax, unresolved references, name collisions and
   unexpected input files with identifying diagnostics. Do not pass an
   unknown construct through merely because its text was tokenized.
3. Preserve source literal/comment boundaries. Use typed table/column
   context for Boolean seed values, not global numeric substitution.
   Preserve H2's default `_pkey` primary-key naming distinction and do not
   assign that suffix to unique constraints.
4. Keep unavoidable dialect overrides narrow and exactly matched, with
   provenance and a removal condition. Stale, unused or mismatched
   overrides must fail. Shared semantic defects identified in H3a remain
   blockers, not automatic overrides.
5. Add explicit PostgreSQL constructors in `domain/schema` using H2's
   `schema.NewForDialect`. Read checked-in assets; do not translate or
   read environment configuration during suite setup. Existing
   `ControllerDDL`, `ModelDDL` and versioned defaults stay SQLite/dqlite.
   Generate initial assets before adding embeds that require them.
6. Add a non-writing `-check` mode to `pgschemagen`. Compare expected
   content and file sets, including missing/extra generated files and
   source/override drift. Separate generated ownership from hand-written
   overrides; never delete arbitrary files. Generation must be stable
   across repeated runs and independent of map iteration or timestamps.

Target CLI; current implementation limits are in the branch record:

```sh
go run ./generate/pgschemagen \
    -dqlite ./domain/schema/dqlite \
    -dest ./domain/schema/postgresql
go run ./generate/pgschemagen \
    -dqlite ./domain/schema/dqlite \
    -dest ./domain/schema/postgresql -check
```

Keep trigger generation before PostgreSQL generation. Integrate with
`go generate ./domain/schema` / `make rebuild-triggers` without introducing
a recursive generator invocation or changing historical `rebuild-ddl`
output. Add a focused freshness check to the existing static schema check;
the separate CI branch still owns PostgreSQL job provisioning.

## H3c: acceptance before shared-suite enablement

Generator tests must cover comments/literals and function bodies, types
and seed data, dependency ordering and cycles, identifier collisions,
registered/version-selected post-patches, unsupported syntax and strict
override matching. Mutate a generated file, remove one, add a stale
generated file, change a Go-built trigger and change a registered patch:
each relevant freshness check must fail, without rewriting the checkout.
Use temporary output trees for these cases.

Run focused schema suites through H2's isolated PostgreSQL helper, not
`DqliteSuite`. For both complete controller and model schemas, verify:

- Fresh application and identical reapplication; persisted version/hash
  chains match the loaded generated sequence. Changed/newer chains and
  incremental upgrades remain explicit errors.
- Real tables, views, seed rows, keys/constraints and trigger functions
  exist and behave correctly, including insert/update/delete effects,
  guard/immutability failures and rollback.
- Timestamp defaults, identity allocation and explicit-ID behavior,
  deferred foreign keys and `_pkey`/unique classification work through
  actual storage and consumers, not only generated-text comparisons.
- Automatically allocated changelog IDs cannot become visible out of commit
  order and cause a watermark reader to miss a later commit. Exercise two
  transactions, rollback/gaps, explicit IDs and independent schemas. A bare
  sequence does not establish this contract; any advisory-lock design needs
  deterministic readiness, cancellation and bounded race/stress coverage.
  On 2026-11-20 the user confirmed that explicit IDs remain supported even
  below a prior watermark; audit their callers and lifecycle before H4 because
  such writes can be missed by an active watermark reader.
- Concurrent application, partial failure and cancellation leave atomic
  bookkeeping and only owned-schema cleanup. No stale-schema sweep.

Planned PostgreSQL suite name: `TestPostgreSQLSchemaSuite` in
`domain/schema/postgresql_test.go`, separate from existing domain suites.
The commands below are future acceptance, not completed H3 evidence.
Supply the approved PostgreSQL connection privately; retain logs through
`tee` with `set -o pipefail` and an outer timeout as in H2.

```sh
# Pure generation plus existing SQLite schema behavior.
JUJU_TEST_DATABASE=sqlite go test -race -p 2 -count=1 -timeout 120s \
    ./generate/pgschemagen ./core/database/schema ./domain/schema/...

# Actual generated schemas on PostgreSQL 16; existing H4 gate stays intact.
JUJU_TEST_DATABASE=postgresql go test -race -p 2 -count=1 -timeout 120s \
    ./domain/schema -run '^TestPostgreSQLSchemaSuite$'

# Existing behavior on real dqlite, using shared bundled dependencies.
JUJU_TEST_DATABASE=dqlite make run-go-tests \
    TEST_PACKAGES='./generate/pgschemagen ./core/database/schema ./domain/schema/...' \
    TEST_ARGS='-race -p 2 -count=1 -timeout 120s'
```

Recompile concurrency-bearing suites with `go test -c -race` and use bounded
`stress` as required by the source rules. Run `gci` on touched non-generated
Go files, scoped lint, compiled `txncheck`, generation freshness and the
relevant schema/static checks. Preserve dqlite source, Go-built patch and
historical release-DDL bytes; record before/after evidence, not just a
clean generated diff.

**H3 exit:** both generated schemas, reproducible generation/check commands,
complete input/override provenance, passing focused PostgreSQL and
SQLite/real-dqlite evidence, and no hidden shared-schema failures. Record
the exact head and limits in `status.md`; stop before H4. H2.1's chosen
sqlair dependency, full SQL-owning coverage, CI timing and production
PostgreSQL support are not established by H3.
