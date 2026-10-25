# Upstream 4.1 review branches

Status: branch proposal ready for dispatch planning. The user approved
upstream-only worktrees (N11) and adding U12 to the existing integer branch
(N12). Other branch names and the initial batch below are recommendations.
No branches/worktrees or code changes were made by this planning session.
Scope is patches suitable for upstream 4.1, not a second PG stack. The
coordinator uses the whole map; an executor needs only the shared rules,
assigned branch row/task requirements and its U*/X* entries. Do not load
every defect or historical handoff into each extraction task.

## Principles

- Start each independent branch from the same refreshed, pinned upstream
  4.1 revision (`B`). Do not base release branches on `pg-poc/*`.
- One review concern per branch; multiple commits are fine when they serve
  that concern. Do not add unrelated fixes merely because they share a
  PoC source commit.
- Upstream-only worktrees are allowed; the PoC retains its single working
  tree. One owner per branch/worktree. No concurrent checkout, rebase or
  reset in another owner's tree. All work remains local until explicitly
  dispatched for implementation/publication.
- Prefer a small active set: one schema owner plus two independent leaf
  fixes is the proposed starting batch. This is not a requirement to
  launch every listed branch at once.
- Distinguish semantic prerequisites from integration conflicts. Shared
  generated files justify sequencing their owners, not automatically
  stacking otherwise independent PRs.

## Baseline and existing branch

Observed during planning:

- PoC base/local `4.1`: `a8591fe1b8`.
- Cached `upstream/4.1`: `04bc64ef54`, 57 commits ahead of that base;
  this is not a freshly fetched upstream-head claim.
- `4.1-sqlite-integers`: `7c96d020d5`, based on the PoC's old upstream base.
  It already contains `6b0f24d11c` (integer FK types, U2) and
  `7c96d020d5` (revision-updater unit counts, U1).

The integer branch already has a worktree at
`~/dev/go/src/github.com/juju/4.1-sqlite-integers`; coordinate its owner
before edits, rather than creating another checkout or force-moving it.
Keep U1/U2 and add U12 as a separate commit (N12); no duplicate U1 branch.
Refresh upstream and check landed equivalents before choosing `B`.

The map below uses commit metadata and targeted leaf diffs, plus a
schema-path/representation inspection. It is not validation of extracted
patches on the eventual upstream base. Confirm each source diff and path
inventory when the task is dispatched.

## Candidate branch map

Names below are proposals except the existing integer branch. Sources are
pinned PoC commits at planning time; use them as evidence, not unquestioned
cherry-pick instructions. Record replacements after PoC restacks.

| Branch | Scope | Source / readiness |
|---|---|---|
| `4.1-sqlite-integers` (existing) | U1/U2 already extracted; add U12 relation-status IDs as a separate commit (N12). | Existing commits above + `ad9d3928c1`; U12 uses U2's parsing helpers. |
| `4.1-binary-key-columns` | U8: raw key material stored as BLOB with matching representations. | `e987e3df60`; schema lane. |
| `4.1-removal-force-boolean` | U9: boolean declaration and 4.0.12 import conversion. | `1513cf1b97`; schema lane. |
| `4.1-model-config-precedence` | U7: authoritative agent stream/version, no duplicate config rows. | `63f0bb3a68`; focused schema/view + modelconfig regression. |
| `4.1-keymanager-insert-id` | U13: return the inserted key ID; reject unknown algorithms without stale-ID reuse. | `0f7d28ec1d`; small independent production fix + regression. |
| `4.1-controller-node-id-text` | U3 and U5: bind unsigned node IDs as decimal text, including fixtures. | `e8ef9d307b`, `03743f82ff`; upgrade fixture half of U5 remains to implement. |
| `4.1-lease-migration-tests` | U4: assert seed success and genuinely exercise held/expired leases. | `bae6008b6e`; adapt missing helper dependency, not the production lease rewrite. |
| `4.1-crossmodel-secret-revisions` | U11: correct multi-unit revision selection with portable SQL. | No finished patch. Dedicated semantic design/regressions first. |

### Schema lane

Use one schema owner to prepare U12, U8 and U9 in sequence, but keep their
reviews independent where possible. U12 genuinely uses U2's `parseID`
helpers and extends its legacy-import fixtures, so it belongs on the
existing integer branch. U8/U9 can each start from `B`; neither needs the
other's behaviour. U7's view-only fix can proceed independently.

Main conflict surfaces to inspect on the selected base:

- U2/U9/U12: `domain/export/types/v4_1_0/model.go`, transformer
  `transforms/to_v4_1_0/{transform.go,deltas.go,deltas_test.go}`, legacy
  import fixtures and model recovery generation.
- U2/U8: `domain/export/types/controller/v4_1_0/controller.go`; U8 also
  changes `domain/schema/storage.go` and controller recovery decoding.
- Remap PoC `domain/schema/dqlite/{model,controller}/...` back to upstream
  `domain/schema/{model,controller}/...`. U7 changes the agent-version
  view patch; U12 relation SQL; U9 cleanup SQL; U8 user/macaroon SQL.

Regenerate from each branch's own schema using the upstream export/import/
recovery tooling (currently `make rebuild-export`). Never copy cumulative
PoC generated outputs containing other fixes. If two reviewed branches
conflict after landing, regenerate on the updated base; do not resolve by
keeping one branch's entire generated file and losing the other change.

Required schema checks: U12 valid/nullable/invalid legacy IDs through
transform/import/export; U8 arbitrary bytes and supported historical
encodings through export/recovery; U9 0/1 conversion, invalid-value rejection
and round trips. Confirm that direct edits still target beta-only schema;
otherwise use the release's incremental patch mechanism, not the PoC's
fresh-database assumption.

### Leaf task requirements

- **Keymanager:** keep this separate from general SQL quotation/equality
  cleanup. Test existing/new keys, unknown algorithms after a prior insert,
  and unchanged associations. No pgx or harness dependency is needed.
- **Node IDs:** add a production regression for a uint64 with the high bit
  set, including max uint64 round-trip as decimal text. U3's source commit
  contains no test. Include the matching changestream/upgrade fixture fixes;
  do not convert the column to signed INT with the integer-schema branch.
- **Lease tests:** `formatTimestamp` was introduced by `f474710187`, the
  broad PoC SQL rewrite. The U4 commit therefore is not standalone. Use an
  upstream-compatible local fixture/helper or approved existing time seam;
  preserve boundary semantics without importing production clock changes.
- **Model config:** carry the shared view fix and regression, remapping the
  schema path back to upstream. No PostgreSQL output or override is needed.
- **Cross-model secrets:** use the
  [dedicated investigation](investigations/crossmodel-secrets.md). Approve
  any/all-consumer and revision-filter semantics before implementing. Keep
  general crossmodel test quotation cleanup out of this branch.

## Suggested first batch

1. **Schema owner:** extend the existing integer branch with U12; then
   prepare key-BLOB and removal-boolean branches one at a time.
2. **Leaf owner A:** keymanager insert-ID correctness.
3. **Leaf owner B:** unsigned controller-node ID binding and fixtures.

When a leaf slot frees, model-config precedence or lease-test reliability
is ready for focused extraction. Cross-model secrets design may proceed
in parallel, but implementation waits for semantic approval. Each new
assignment gets its own branch/worktree and bounded context; do not let
owners sweep unrelated portability fixes into these branches.

This limits simultaneous integration surfaces without forcing reviewers
to wait for one monolithic schema/SQL PR. Broad test-literal cleanup comes
later because its 49-file footprint intersects many functional owners.

## Follow-on portability reviews

Do not bundle these with concrete correctness fixes merely to reduce PR
count. They can land in 4.1 if independently justified, but are not all
behaviour-neutral or ready to extract.

| Proposed branch | Scope / source | Review requirement |
|---|---|---|
| `4.1-changestream-latest-row` | X16, `48f0c5e1a9` | Prove the full returned row comes from the latest ID; high-water marks, coalescing and pruning remain correct. Source commit adds no tests; add focused regressions and race/stress evidence. |
| `4.1-blockcommand-order` | U10/X31, `a8301140de` | Explicit approval of insertion-order to type-order change; regression is included. |
| `4.1-operation-query-portability` | X34/X33, `18886acfdf`, `e2258798fe` | Separate commits for boolean filtering and size estimates; preserve filtering and pruning thresholds. Coordinate X24 edits in the same package. |
| `4.1-objectstore-conflicts` | X36, `d658b402a6` | Review the reused-UUID error change, not just SQL syntax; source includes a regression. |
| `4.1-sql-test-literals` | U6/X6, `8a60208176` | Broad 49-file cleanup: apply after overlapping functional fixes, or reserve explicit files for their owners. Revalidate fixture meaning. |

Other shared fixes need narrow task scopes before dispatch: X3/X4 upsert
qualification (`ff382936b3`), X24 CTE membership (`fda7add768`), X7 model
IIF (`9499d95ca6`), X37 equality and X38 removal join scope (not yet fixed).
The initial `f474710187` combines syntax, aggregation and lease time changes;
never export it wholesale as a mechanical cleanup.

Likewise, X18 (`01316d98fe`, `bccb8d37b4`) changes conflict handling across
several domains. Split by operation and preserve retry/idempotency/error
contracts, especially the external-user existence-check path. No blanket
assumption that SERIALIZABLE turns every unique conflict into a retry.

Test-only user quoting/date/upsert conversions can be later portability
patches; coordinate their files with active functional owners. PostgreSQL
helper APIs, driver rewrites, harness skips, config/workers, schema selector,
PG overrides and benchmark tooling are not part of these 4.1 branches.

## Dispatch and validation

Each assignment supplies `B`, one named branch/worktree, allowed paths,
source commit(s), required adaptation, regression cases and test commands.
Stop on unplanned dependencies or shared-file ownership conflicts.

Starting package scopes: keymanager `domain/keymanager/state`; node IDs
`internal/database`, `domain/changestream/state`, `domain/upgrade/state`;
lease fixtures `domain/lease/state`; model config `domain/modelconfig/state`
and relevant schema tests; cross-model secrets
`domain/crossmodelrelation/state/model` plus the consumers identified by
its design task. Run only affected selectors initially, then the package
suite before submission. Schema-lane tasks additionally need the relevant
export/import/round-trip suites, not just successful DDL generation.

Acceptance is against upstream without PostgreSQL: unset
`JUJU_POSTGRESQL_TEST_URL`; run affected package tests, real dqlite checks
where needed, and required race/stress coverage. Schema changes must trace
writers -> storage -> export -> legacy import -> consumers, exercising
supported representations in both directions. Regenerate only with the
upstream tools and verify no unrelated generated changes. Follow target
repository formatting/lint rules and PR template.

One coordinator updates the plan/register and source-to-upstream mapping;
parallel agents return reports rather than editing shared planning files.
Concurrent tests still need a host resource budget. They must not overlap
shared PostgreSQL runs or modify shared Juju/controller state.

## Landing and PoC reintegration

1. Review/land independent branches normally into upstream 4.1. Do not
   merge them into each other just to create a new release mega-stack.
2. Track each U*/X* item through PoC source, extraction commit, upstream
   PR/merge commit and any intentional semantic difference. Splits or
   squashes mean patch-ID equality is insufficient.
3. At an agreed checkpoint, freeze PoC edits, preserve the old stack tips
   and refresh/pin the new upstream 4.1 base containing landed changes.
4. Rebase/reconcile the PoC stack as one controlled operation. Remove
   duplicate shared hunks, not entire mixed commits that still contain PG
   work. Preserve stage refs and investigate conflicts or skipped commits.
5. Reapply/regenerate the PoC schema move, PG output and overrides against
   the new shared schema. Recheck export/import adjustments and remove only
   overrides whose purpose really disappeared. Verify adjacent ancestry.
6. Build the top and rerun affected dqlite/PG tests plus D42's runtime-entry
   gate. Update base/status/mapping only with recorded evidence.

Do not rebase the PoC after every leaf commit. Prefer one reconciliation
per coherent landed batch; the first batch should include the existing
integer work and a small set of independent fixes. An optional integration
branch for unmerged PRs is not the upstream baseline and needs a separate
decision if wanted.
