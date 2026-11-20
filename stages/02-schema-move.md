# Stage 02: source ordering and schema layout

Branch `pg-poc/02-schema-move`, on `pg-poc/01-provision`.

Current result, 2026-11-20: source ordering, layout and strict PostgreSQL
generation are implemented and validated locally through Stage 9. The
earlier planning-only session and original move-only verification remain
historical evidence, separate from this implementation dispatch.

## 02-04-schema-alignment

Status: **implemented and validated locally at `09=893eb9105d`;
publication is not authorized**.

Owner/session: `06cbcf48-2444-4d6e-b2c8-0dc628c0b465`, working only in
`../postgresql-poc`. The user dispatched this task and approved fresh,
uniquely named test namespaces on the PostgreSQL 16 endpoint in the private
URL file. Existing databases, controllers and historical schemas are not
approved cleanup targets. No push or publication is authorized.

The clean baseline matched all ten recorded tips. Recovery tags
`pg-poc-before-schema-alignment-a1a9610167/00-base-fixes` through
`/09-bootstrap` preserve them. The upstream base remains `d380e5e4fb`.
Local Stage 2 is rebuilt as:

- `dda958e1a5`: adapted source ordering from pinned `b622931495`.
- `24450383b5`: mechanical layout from pinned `9c8f98cd16`.

Stage 3 replayed to `5a19411afd`; Stage 4's strict generator is committed
at `5832c5c9b8`. Stages 5–9 are restacked through `893eb9105d`, with every
adjacent stage's ancestry verified. D56's charm timestamp and insertion
order remain in `0110-workload-core.sql`. Exposure definitions follow
`application_endpoint` in `0210-application.sql`; the existing Stage 5
`v_model_config` correction is in `0290-agent-version.sql`. Generated
conflicts were resolved by regeneration, not whole-file replacement.
No upstream worktree, base, experiment branch or Stage 9 checkpoint scope
was changed.

### Implementation evidence, 2026-11-20

Baseline and ordering-parent SQLite snapshots captured every assembled patch,
schema object definition and seed row. Controller: 292 objects, 94 tables;
model: 792 objects, 250 tables. Objects and seed rows match apart from the
known trailing space in one `v_secret_permission` comment and the naturally
generated `object_store_backend` insertion timestamp; its other seed values
and timestamp default definition match. Source order changes patch counts
from 61/156 to 63/158. The newer upstream `v_model_config` correction was
not imported into Stage 2; the existing Stage 5 correction must replay.

The layout parent/result have byte-identical assembled controller/model
patch arrays, including Go-built triggers: 63/158 patches. All 104 moved
assets are exact renames. Patch hashing is unchanged, so hashes are unchanged
relative to the ordering parent. `go generate ./domain/schema/...` left no
unstaged diff. Both post-patch registries are empty and the base remains
`4.1-beta3`; this work does not support existing beta database upgrades.

Race-enabled SQLite and real-dqlite selections passed for schema,
agentbinary, schema core and ddlgen after ordering, and schema, agentbinary,
modelmigration and ddlgen after layout. Each invocation used an outer
150-second bound and `-v -race -count=1 -timeout 120s`; real dqlite used
`make run-go-tests`, layout selections used `-p 4`. The IDE build returned
success with limited diagnostics. `gci` was unavailable on PATH; the pinned
`go run github.com/daixiang0/gci@v0.13.7 write` command formatted touched
non-generated Go files. `git diff --check` passed.

Evidence is under this session's `files/`: `baseline-stage-one/`,
`baseline-top/`, `ordered-stage-two/`, `layout-stage-two/`,
`stage-two-*.log`, `alignment-old-tips.txt`,
`alignment-later-source.diff` and `alignment-original-postgresql.tar`.
The subsequent generator, replay and final-tip evidence follows.

### Final integration and acceptance, 2026-11-20

Stage 4 removes dependency relocation: validation runs after translation
and exact-match overrides, checking statement-level dependencies without
moving views, stripping FKs or appending repair constraints. Fixtures
cover prior/composite/implicit keys, valid self-reference, same-/cross-patch
forward references, dangling dependencies, missing keys and columns,
partial/expression indexes, views, functions, triggers and cycles.
Invalid generation returns no output; valid input preserves source order.
This uses the existing SQL lexer, not a general PostgreSQL grammar parser.

Generated base names preserve source numbers and gaps; Go-built trigger
names continue after the maximum source number, including distinct patches
watching the same table. Exact override purposes and old-to-new names are
recorded in [Stage 4](04-schemagen.md#current-override-inventory).
Replayed `62abb7d31c` removes only the obsolete offer relation-status text
comparison from the consolidated `0360-cross-model-relation.diff`, retaining
its duplicate-FK-name correction.

Final snapshots match the original top's 292 controller objects/94 tables
and 801 model objects/252 tables. The only differences are the naturally
generated controller seed timestamp, the pinned trailing-comment-space
cleanup, and an explanatory comment on the preserved `v_model_config`
predicate. All other definitions and seed values match. Final assembled
patches are 63 controller and 160 model, versus the original top's 61/158.
Historical release assets, post-patch registries and hashing code are
unchanged; no live old source-path consumers remain. Repeated final
generation produces no tracked or untracked changes, and freshness tests
pass.

At `893eb9105d`, the affected generator/schema/agentbinary/modelmigration/
modelimport/export/recovery selection passed with `-race` on SQLite and
real dqlite. Both complete PostgreSQL schemas passed fresh application,
reapplication, persisted versions/hashes, changed-hash rejection, rollback,
keys, seeds, views, triggers and D13 commit visibility/explicit-ID/gap
checks. A deliberately unreachable endpoint produced the required
connection failure, proving actual PostgreSQL selection. The server was
PostgreSQL 16.15; zero test namespaces existed before or after validation.

The user approved both final 142-package sweeps with server exclusivity
for PostgreSQL. Both used `GOMAXPROCS=2`, `-json -v -race -count=1 -p 4
-parallel 1 -timeout 40m`, with an outer 45-minute safety bound.
The manifest reproduces Stage 6's selection, SHA-256
`d08b0919855142a445d1477fe78a2593c638d272668d2e7cbd714e9250cf82ed`.

| Backend | UTC interval, including startup | Selected packages | Leaf outcomes |
|---|---|---|---|
| Real dqlite | 14:31:12–14:38:46 (7m34s) | 140 passed; 2 without standalone tests | 5,557 passed; 82 skipped |
| PostgreSQL | 14:31:12–14:47:24 (16m12s) | 140 passed; 2 without standalone tests | 5,428 passed; 189 skipped |

Neither sweep failed or reported a race. The two test-support packages are
`internal/changestream/testing` and `internal/database/testing`; skips are
not passes. Existing PostgreSQL skips, including 105 schema methods, remain.
The manifest still excludes `domain/modelimport`,
`domain/modelimport/state/model`, `internal/migration` and
`domain/recovery/state`; D40 is not broadened or removed. Required
SQLite/dqlite import/export/recovery fixtures were covered separately.

Race-enabled schema binaries completed 10 stress runs each on SQLite and
real dqlite, and 10 serial PostgreSQL controller/changelog runs, all without
failures. Each stress invocation had a 31-second outer bound and a
30-second per-process bound. PostgreSQL stress ran after its exclusive
sweep. `GOMAXPROCS=2 go build -p 2 ./...`, scoped golangci-lint v2.14.0
(zero issues), txncheck for all four changed package areas, gci, CI's
SQLFluff 3.0.7 source-SQL command and whitespace checks passed.
`make pre-check` ran without `STATIC_ANALYSIS=1`, so it is not a new full
configured-static-analysis pass. The IDE build also returned success with
limited diagnostics.

Final evidence is in the same session's `files/`: `final-top/`,
`final-sweep-packages.txt`, `final-{dqlite,postgresql}-sweep.jsonl`,
`final-{dqlite,postgresql}-summary.json`, `final-sqlite.log`,
`final-dqlite-consumers.log`, `final-*-stress.log`, `final-build.log`,
`final-scoped-lint-pinned.log`, `final-txncheck-*.log`,
`final-sql-lint-pinned.log`, `final-generation*.log` and
`postgresql-final-{before,after}.log`.

At handoff, the planning changes were uncommitted and another session's dirty
`UPSTREAM-FIXES.md` edit was left untouched. The coordinator subsequently
linked this record from the register. No publication, live bootstrap,
benchmark, HA work or unrelated checkpoint redistribution was performed.

Outcome: the PoC uses the same source organization as the current upstream
breakout and generates PostgreSQL DDL in that explicit order. Preserve the
existing PoC's schema/query corrections, runtime behavior and backend
selection. This is one integration task with separately reviewable
source-order, layout and PostgreSQL-generation commits.

### Branches, prerequisites and references

- One implementation owner in `../postgresql-poc`; rebuild Stage 2 on
  `pg-poc/01-provision`, make generator changes on
  `pg-poc/04-schemagen`, and reconcile through `pg-poc/09-bootstrap`.
  Keep all stages in the existing linear stack. No extra PoC worktree.
- Current baseline: upstream `d380e5e4fb`, Stage 2 `db349a2433`, Stage 4
  `3cec77ac4a`, top `a1a9610167`. Recheck all ten tips, working-tree
  changes and ownership before dispatch; preserve recovery refs and
  outstanding work before rewriting any history.
- Read [PLAN.md](../PLAN.md#poc-schema-alignment-2026-11-20),
  D10/D12/D13/D15/D43/D55 in [DECISIONS.md](../DECISIONS.md),
  [Stage 4](04-schemagen.md), and the stack, generation and validation
  sections of [OPERATIONS.md](../OPERATIONS.md). Follow the code
  checkout's root/nested guidance and test instructions.
- Reference source ordering:
  [`4.1-schema-dependency-order`](../upstream/4.1-schema-dependency-order/status.md),
  `b622931495f0fb3a2242418c82e2fcb152183354`.
  Reference layout:
  [`4.1-schema-layout`](../upstream/4.1-schema-layout/status.md),
  `9c8f98cd16289cb0e17b9be9ac587d59a683b285`, directly on ordering.
  Pin the reviewed source heads when dispatched; later review changes
  require explicit reconciliation, not an automatic moving target.
- Those heads have upstream parent `59dcd6d951`, newer than the PoC
  base. Use their ordering/layout changes, not their entire source trees
  or ancestry. A base refresh is not part of this task. Do not modify
  either upstream worktree or the independently owned PostgreSQL harness.
- Recheck release history and registered post-patches. Ordering changes
  the hash chain; this task does not support in-place upgrades of existing
  beta PoC databases. Obtain explicit targets/approval before recreating
  disposable databases or performing live bootstrap/teardown.

### Bounded implementation

1. Capture the baseline assembled patches, schema objects, seed data and
   hashes, together with later PoC schema corrections and active overrides.
   Map each definition from the old files to the upstream file/statement
   order. Compare actual definitions, not just Git rename detection.
2. Rebuild Stage 2 as two concerns in upstream order: shared source-order
   cleanup, then mechanical layout relocation. Preserve the current-base
   SQL semantics when adapting the reference patch; do not incidentally
   import newer upstream query changes. Retain provenance to both source
   heads and record adaptations honestly; an adapted patch is not an
   unchanged D43 cherry-pick.
3. Use `domain/schema/dqlite/{controller,model}/` for source SQL, triggers
   and release assets, and keep PostgreSQL beside it. Match upstream base
   filenames, numbering gaps, file splits and within-file ordering.
   Bring across the statement-level dependency regression guard, including
   matching FK keys, view/trigger dependencies and valid self-reference.
   Keep seed-write and trigger side effects in their intended order.
4. Wire every applicable path consumer: `domain/schema` embeds, patch
   discovery, generation directives and post-patch registries;
   `domain/agentbinary` architecture lookups/tests; `domain/modelmigration`
   references/tests; `generate/ddlgen`; workflow filters and SQL lint;
   schema guidance. Use APIs present at the owning stage; do not add
   later upstream helpers or duplicate patch registries just to copy a
   newer implementation.
5. Replay Stages 3 through 9 with definition-aware conflict resolution.
   Preserve Stage 4's D56 timestamp/insertion-order changes, Stage 5's
   shared schema fixes, import/export deltas and subsequent fixture
   corrections. Never choose a whole upstream or generated file over
   those changes. Record an old-to-new source/override mapping.
6. On Stage 4 replace PostgreSQL dependency relocation with strict
   source-order validation after translation and overrides. Reuse relevant
   parser/dependency helpers, but remove the production relocation path:
   no moved views, stripped FKs or added late constraints to repair an
   invalid source order. Report namespace, source patch and offending
   dependency; reject same-file as well as cross-file forward references,
   missing target keys, dangling references and multi-object cycles.
   Validate translated trigger/function dependencies as well.
7. Preserve base SQL filenames in generated assets; `patchNames` currently
   replaces their numbers with patch indexes. Keep one output per assembled
   patch and deterministic, collision-free identities for Go-built trigger
   patches without adding a second registry. Remap `.diff` overrides by
   moved definition, rebaseline `-auto.sql` from raw translation, and
   regenerate all PostgreSQL assets. Keep exact hunk matching and genuine
   dialect translations; remove ordering-only overrides/repairs only once
   their purpose is demonstrably gone. Existing temporary shared-defect
   overrides retain their finding/removal obligations under D55.
8. Update generator/schema guidance and affected planning references,
   including Stage 4's override inventory. Historical release DDL bytes
   and captured acceptance results remain unchanged. The current
   `.PATCH.sql` rejection is not broadened into upgrade support here;
   stop if new registrations make that limitation relevant.

### Acceptance and stop conditions

The checks below remain the acceptance contract; their revision-specific
results and retained coverage gaps are recorded above.

- Compare the source-order-only result with its prior baseline: table,
  column, constraint, index, view and trigger definitions and seed rows
  must match except for explicitly explained comment/layout differences.
  Verify intermediate dependency validity and write/trigger effects,
  not just final object counts. The layout-only commit must preserve
  assembled statement bytes/order and hashes relative to its ordering
  parent, including Go-built triggers.
- At the final top verify every stage's ancestry, preserved later
  corrections, exact output/source file mapping, complete patch counts and
  deterministic regeneration. Generate twice with no second diff;
  freshness must detect missing/extra output, stale overrides and source
  drift. Do not hand-edit generated SQL.
- Add strict-generator fixtures for prior references, self-references,
  same-/cross-patch forward FKs and keys, views, triggers/functions,
  dangling references and cycles. Invalid order must fail without output
  relocation; valid ordered input must preserve statement order.
- Run affected generator/schema/consumer suites with `-race` on SQLite
  and real dqlite using the checkout's `make run-go-tests`. Initial
  selection: `./generate/pgschemagen ./generate/triggergen ./generate/ddlgen
  ./domain/schema/... ./core/database/schema ./domain/agentbinary/...
  ./domain/modelmigration/... ./domain/modelimport/... ./domain/export/...
  ./domain/recovery/...`. Include tests/consumers of all conflict-resolved
  schema definitions; retain the D40 PostgreSQL import/export deferral.
- On PostgreSQL 16 apply both complete schemas to fresh unique test
  namespaces; check identical reapplication, persisted versions/hashes,
  changed-hash rejection, rollback, keys, views, seed data and trigger
  effects. Exercise changelog commit visibility, explicit IDs and rollback
  gaps without altering D13. Prove the backend actually opened, not just
  that a URL was configured.
- Run bounded race-enabled stress for changed concurrency-bearing
  consumers, required formatting, scoped lint/txncheck and relevant SQL
  checks. Run the final full build and agreed Stage 6 selections once per
  backend after complete integration, with existing skips/deferred
  coverage explicit. Record budgets, exact revisions and logs under
  OPERATIONS.md; focused passes do not replace final-stack acceptance.
- Stop for unexpected ownership/dirty conflicts, uncertain semantic
  changes, genuinely cyclic dependencies requiring redesign, unsupported
  registered post-patches, release-policy conflicts or missing database
  approval. Do not introduce new skips, defer constraints, relax strict
  validation or import another owner's pending fixes to make it pass.
- Record source provenance/adaptations and final validation in this stage,
  Stage 4 and UPSTREAM-FIXES.md, then update PLAN.md's tips/status.
  Stop after alignment and handoff. Do not redistribute unrelated Stage 9
  checkpoints, run benchmarks/HA or publish refs/PRs without separate
  approval.

## Original move-only implementation

- Goal: separate the dqlite DDL so a PostgreSQL schema can sit beside it.
  Mechanical, with no behaviour change.
- Steps:
  1. `git mv domain/schema/{controller,model}` (SQL, triggers, release
     DDL) under `domain/schema/dqlite/`, keeping the existing Go API
     (`schema.ControllerDDL()`, `schema.ModelDDL()`) as thin wrappers.
  2. Update `go:generate` directives, `generate/triggergen`, embed paths and
     `domain/schema/AGENTS.md`.
- Verification: `go generate ./domain/schema/...` produces no diff;
  `go test ./domain/schema/... ./core/database/...` passes; schema hashes are
  unchanged.
- Status: in review. Committed on `pg-poc/02-schema-move` (one `refactor`
  commit); verification passed.
- Restart notes:
  - Done 2026-10-23: `git mv` of `domain/schema/{controller,model}` to
    `domain/schema/dqlite/`; embed paths, `go:generate` destinations,
    trigger package imports, `generate/ddlgen`, `.github/workflows/ddl.yml`
    and `context-tests.yml`, comments and `domain/schema/AGENTS.md` updated.
    The Go package stays `domain/schema` with `ControllerDDL()` and
    `ModelDDL()` unchanged, so no callers changed.
  - `go generate ./domain/schema/...` left no diff; `go test
    ./domain/schema/... ./core/database/... ./domain/agentbinary/...
    ./domain/modelmigration/...` passes (the schema suites cover hashes).
  - Next: user review.
- Findings (for later stages):
  - Stage 4 should add `domain/schema/postgresql/` beside `dqlite/`; the
    `schema` package (`controller.go`, `model.go`, `patch.go`) still owns
    the dqlite embed and `readPatches`, and will need a backend selector
    (resolved by D11).
  - Historic `.ddl` release files contain stale path comments; left as is.
  - The build errors and `gofmt` noise seen during this stage were a Go
    1.24 `gofmt` on `PATH` plus ungofmt-ed generated trigger files; both
    are resolved by [Stage 00](00-base-fixes.md).
