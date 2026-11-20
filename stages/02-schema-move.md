# Stage 02: source ordering and schema layout

Branch `pg-poc/02-schema-move`, on `pg-poc/01-provision`.

Current direction, 2026-11-20: align with the upstream dependency-order
and layout breakouts under the task below. The original move-only
implementation and its verification remain historical evidence; they do
not establish acceptance of the new source order or strict generator.
Planning only was authorized in this session. No code, refs or databases
were changed.

## 02-04-schema-alignment

Status: **defined; awaiting implementation dispatch**.

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

All checks below are **required future evidence**, not completed results.

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
