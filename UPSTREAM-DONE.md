# Upstream work completed and rationale

This records implemented upstream work, decisions and validation, including
changes still in review. **Implemented is not landed or reintegrated.**
The landed/in-review inventories reflect a 2026-11-05 GitHub snapshot,
with #23728's merge recorded on 2026-11-06. Validation and publication
narratives retain their recorded dates and revisions; they are not new
checks of the latest heads.

- [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md): remaining work and acceptance checks.
- [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md): finding register, source provenance,
  current upstream disposition and eventual PoC reconciliation evidence.
- [UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md): reusable reintegration
  procedure. Landing upstream does not establish PoC reconciliation.

The coordinator maintains this completed-work summary. New execution history,
decisions, validation and pending actions belong in each branch's linked
`status.md`; finding mappings and PoC reconciliation remain in the register.
Historical heads and test runs do not establish coverage of later revisions.

For new work, use the plan's
[SQL/schema acceptance contract](UPSTREAM-PLAN.md#purpose-and-sqlschema-acceptance)
(D55), not a historical implementation as a prescribed design. Legitimate
representation conversions remain valid: U3 stores the full unsigned
64-bit node-ID range as decimal text, which a signed SQLite integer cannot
represent. That is distinct from U21's pending correction of integer
relation-key storage; not every Go numeric type implies an integer column.

## D55 merged implementation assessment (2026-11-06)

Design-alignment review against upstream `148e09f015` and the actual merged
SQL/schema changes, with GitHub merge states checked on 2026-11-06.
This did not rerun tests, functional QA or CI, and is not a full bug audit.
Open-branch rework/merge holds and pending follow-ups belong in
[UPSTREAM-PLAN.md](UPSTREAM-PLAN.md#d55-implementation-review-2026-11-06).

| Merged branch / PR | D55 assessment / follow-up advice |
|---|---|
| [`4.1-sqlite-integers`](upstream/4.1-sqlite-integers/status.md), #23680 (U1/U2/U12) | Aligned: correct grouping at application level and change incorrectly declared lookup/FK IDs to INT. Legacy parsing belongs at import. No D55-driven follow-up indicated. |
| [`4.1-binary-key-columns`](upstream/4.1-binary-key-columns/status.md), #23682 (U8) | Aligned: raw key bytes become BLOBs and the obsolete TEXT storage exceptions are removed. Matching export/recovery changes are necessary, not a workaround for the old schema. No D55-driven follow-up indicated. |
| [`4.1-removal-force-boolean`](upstream/4.1-removal-force-boolean/status.md), #23690 (U9) | Aligned: BOOLEAN storage, bool exports and validated legacy 0/1 import conversion. Do not undo this by converting normal writes to integers. |
| [`4.1-controller-node-id-text`](upstream/4.1-controller-node-id-text/status.md), #23685 (U3/U5) | Aligned exception: dqlite node IDs span uint64, beyond signed SQLite integers. Decimal TEXT is intentional storage; formatting it at the boundary is correct. Do not propose an INT-schema follow-up based on U21's different domain contract. |
| [`4.1-keymanager-insert-id`](upstream/4.1-keymanager-insert-id/status.md), #23684 (U13) | Aligned: tie the ID to the actual insert and reject an unknown algorithm instead of trusting stale LastInsertId state. A follow-up is reasonable and already open as #23756 for the separate dqlite RETURNING error-response defect. Keep the original correctness guard. |
| [`4.1-lease-migration-tests`](upstream/4.1-lease-migration-tests/status.md), #23686 (U4) | The silent setup-error defect is fixed, but the merged helper still uses `datetime('now', ?)` with SQLite modifiers. It does not bind computed timestamps as earlier documentation claimed. An independent fixture-portability follow-up is [queued at low priority](upstream/lease-migration-fixture-timestamps/status.md#todo-lease-migration-fixture-timestamps-low-priority); no storage-type rewrite is indicated. |
| [`4.1-changestream-latest-row`](upstream/4.1-changestream-latest-row/status.md), #23691; `4.1-dbrepl-latest-row`, #23730; `4.1-local-secret-revisions`, #23720 | Aligned: explicit latest-ID/revision selection and joins/filters replace bare aggregate columns. No D55-driven redo. U19's other secret queries remain separate work; #23720 did not claim to fix them all. |
| [`4.1-objectstore-conflicts`](upstream/4.1-objectstore-conflicts/status.md), #23694; `4.1-agentbinary-conflict`, #23728 | Aligned: SQL resolves intended uniqueness conflicts without continuing after a statement error. UUID/content conflicts and immutable registrations retain their domain meanings. Shared helper/refactoring and error mapping do not substitute for the SQL fix. |
| [`4.1-blockcommand-order`](upstream/4.1-blockcommand-order/status.md), #23692 | Aligned: explicit block-type ordering replaces implicit rowid ordering. This is the agreed ordering change, not a conversion workaround. |
| [`4.1-sql-standard-syntax`](upstream/4.1-sql-standard-syntax/status.md), #23695; `4.1-sql-test-literals`, #23696 | Aligned: equality/join formulation and literal quoting are fixed in SQL itself, including production paths. No D55-driven follow-up indicated. |
| [`4.1-dbaccessor-pool-cleanup`](upstream/4.1-dbaccessor-pool-cleanup/status.md), #23726 (U14) | Separate resource-ownership defect, not an SQL/schema concern. No D55-driven rework or follow-up indicated. |

Do not generalize this assessment into PostgreSQL acceptance, current QA
completion, or permission to change merged representations without a
separately demonstrated defect.

## Landed upstream fixes

| Branch / PR | Finding | Work and rationale |
|---|---|---|
| [`4.1-sqlite-integers`](upstream/4.1-sqlite-integers/status.md), #23680 | U1/U2/U12 | Correct application unit counts and integer lookup/relation-status declarations. SQLite coercion hid mismatched TEXT IDs; legacy string IDs are converted and exports use integer types. |
| [`4.1-binary-key-columns`](upstream/4.1-binary-key-columns/status.md), #23682 | U8 | Store activation, bakery and macaroon root keys as BLOBs, with matching export/recovery and legacy compatibility. Raw random bytes are not text. |
| [`4.1-keymanager-insert-id`](upstream/4.1-keymanager-insert-id/status.md), #23684 | U13 | Obtain the key ID from the inserted row. A zero-row insert for an unknown algorithm must not reuse a prior `LastInsertId` and associate an unrelated key. |
| [`4.1-controller-node-id-text`](upstream/4.1-controller-node-id-text/status.md), #23685 | U3/U5 | Bind decimal text to the TEXT node-ID column, including high-bit and max-uint64 values; correct changestream and upgrade fixtures. Do not narrow IDs to signed INT. |
| [`4.1-lease-migration-tests`](upstream/4.1-lease-migration-tests/status.md), #23686 | U4 | Assert seed success and seed held/expired leases with SQLite datetime expressions so tests exercise actual rows rather than silently passing on empty setup. Computed timestamp binding remains a possible separate portability follow-up, not part of the merged implementation. |
| [`4.1-removal-force-boolean`](upstream/4.1-removal-force-boolean/status.md), #23690 | U9 | Match boolean writers/readers with BOOLEAN storage and bool exports; convert legacy 0/1 and reject other integers. |
| [`4.1-changestream-latest-row`](upstream/4.1-changestream-latest-row/status.md), #23691 | X16 | Explicitly select the latest change ID and join to its row, avoiding ungrouped columns beside MAX. This does not fix the separate dbrepl query. |
| [`4.1-dbrepl-latest-row`](upstream/4.1-dbrepl-latest-row/status.md), [#23730](https://github.com/juju/juju/pull/23730) | X16 | Apply explicit latest-ID selection and row join-back to the separate dbrepl change-stream query, preserving grouping, bounds, ordering and displayed fields. Merged 2026-11-04. |
| [`4.1-dbrepl-output`](upstream/4.1-dbrepl-output/status.md), [#23757](https://github.com/juju/juju/pull/23757) | X16 follow-up | Share change-log/change-stream rendering without changing either query or output. Merged 2026-11-09. |
| [`4.1-local-secret-revisions`](upstream/4.1-local-secret-revisions/status.md), [#23720](https://github.com/juju/juju/pull/23720) | U11 follow-up | Independently select latest local consumed-secret revisions in two query methods, with empty/multiple-secret/query-round-trip regressions and redundant deduplication removed. Merged 2026-11-04; the five-query latest-revision task remains separate. |
| [`4.1-dbaccessor-pool-cleanup`](upstream/4.1-dbaccessor-pool-cleanup/status.md), [#23726](https://github.com/juju/juju/pull/23726) | U14 | Close unretained pools on setup and reopen failures while preserving caller/worker ownership of retained pools. Merged 2026-11-04. |
| [`4.1-cte-membership-subqueries`](upstream/4.1-cte-membership-subqueries/status.md), [#23750](https://github.com/juju/juju/pull/23750) | X24 | Replace eleven SQLite membership shorthands with explicit single-column subqueries, preserving scoped behavior. Merged 2026-11-13 as `20a76d44fe97e2f9b5a1c5bcf01d9b2cf19d60d1`, final head `67daa565a6`; author reported real-dqlite follow-up validation and manual QA. PoC reconciliation is pending. |
| [`4.1-agentbinary-conflict`](upstream/4.1-agentbinary-conflict/status.md), [#23728](https://github.com/juju/juju/pull/23728) | U16/X18 | Use `ON CONFLICT (version, architecture_id) DO NOTHING` and compare retained registration in both databases, preserving idempotency, immutability and transaction usability. Merged 2026-11-06. |
| [`4.1-lxd-default-bases`](upstream/4.1-lxd-default-bases/status.md), [#23756](https://github.com/juju/juju/pull/23756) | U13 follow-up, test diagnostics and dqlite CI | Preserve SSH-key setup diagnostics and failure status; avoid corrupted retryable dqlite `RETURNING` errors in keymanager, with a sentinel and broader real-dqlite CI. Merged 2026-11-11. |
| `4.1-into-4.2`, [#23775](https://github.com/juju/juju/pull/23775) | Mainline integration | Merge the recorded 4.1 source into main; merged 2026-11-10. Later 4.1 commits and mainline `IIF` cleanup remain separate follow-ups. |
| [`4.1-blockcommand-order`](upstream/4.1-blockcommand-order/status.md), #23692 | U10/X31 | Replace implicit rowid order with explicit block-type order. This is an agreed observable ordering change, not a syntax-only rewrite. |
| [`4.1-objectstore-conflicts`](upstream/4.1-objectstore-conflicts/status.md), #23694 | X36 | Use one conflict clause and distinguish UUID reuse from content already stored; details below. |
| [`4.1-sql-standard-syntax`](upstream/4.1-sql-standard-syntax/status.md), #23695 | X37/X38 | Standard equality and an explicit CROSS JOIN to the one-row fulfilment aggregate. Cover unused/shared-key deletion and zero-fulfilment detach info. A third commit changes storage lifecycle trigger equality. |
| [`4.1-sql-test-literals`](upstream/4.1-sql-test-literals/status.md), #23696 | U6 | Replace double-quoted string literals in fixtures and production SQL with standard single-quoted strings, preserving meaning. |

PR #23680 merged on 2026-10-30 at head `0e9be8ddd7`. Its review follow-up
added full-row conversion coverage, invalid-ID classification, explicit
view row counts and schema-evolution documentation. Race tests, 52 stress
iterations and targeted lint passed; live QA was not run. Details:
[integer review evidence](upstream/4.1-sqlite-integers/reply-review.md).

PR #23691 merged on 2026-10-30 as `ece31c289c6a`; PR #23694 merged that day
as `75ddfcbaf0b3`. Other landed heads and historical source mappings remain
in [the register](UPSTREAM-FIXES.md#upstream-review-status).

PR [#23730](https://github.com/juju/juju/pull/23730) merged on 2026-11-04
as `f7b596bf5cb9b51aa44ec000d09b9f495aac29ad`, at head `49a1bdf808`.
The latest-row SQL task is complete. The separate duplicated-output
refactor subsequently landed in #23757. No PoC integration was performed;
dbrepl remains outside the PostgreSQL PoC scope.

PR [#23720](https://github.com/juju/juju/pull/23720) merged on 2026-11-04
as `edb2aef5b9f1e4cf7c5ceb890fd137d922048c30`, at head `ee07d7ccb5`.
PR [#23726](https://github.com/juju/juju/pull/23726) merged that day
as `5259edff4329ca4a1de80d77e0d614b832d4ad72`, at head `ee55bab5f4`.
Both tasks are complete upstream and were reconciled into the PoC on
2026-11-05 against base `5259edff43`, with default-backend validation only.
PostgreSQL validation remains outstanding; see
[the reconciliation record](UPSTREAM-FIXES.md#poc-reconciliation-onto-5259edff43-2026-11-05).

PR [#23728](https://github.com/juju/juju/pull/23728) merged on 2026-11-06
as `32be283d029d39e83027487899add9b94c426f86`, at head `2618d146da`.
Its final follow-up shares registration state between the model and
controller implementations. The earlier PoC cherry-picks were replaced by
the merged implementation in the 2026-11-11 rebase; validation remains
incomplete, as recorded in UPSTREAM-FIXES.md.

The equality change to the storage lifecycle triggers changes the schema
patch hash: model databases created by an earlier 4.1 beta fail that hash
check. Released 4.0.15 DDL is unchanged. Do not describe this as preserving
all earlier-beta database compatibility.

## 4.1 to main integration

See the [branch record](upstream/4.1-into-4.2/status.md#41-to-main-integration).

## LXD diagnostics and keymanager retryability

See the [branch record](upstream/4.1-lxd-default-bases/status.md#lxd-diagnostics-and-keymanager-retryability).

## Dbrepl output refactor

See the [branch record](upstream/4.1-dbrepl-output/status.md#dbrepl-output-refactor).

## Landed dependency follow-up (2026-11-09)

See the [branch record](upstream/dqlite-returning-errors/status.md#landed-dependency-follow-up-2026-11-09).

## Implemented upstream, still in review

| Branch / PR | Finding | Implemented scope and rationale |
|---|---|---|
| [`4.1-model-config-precedence`](upstream/4.1-model-config-precedence/status.md), #23683 | U7 | Exclude agent stream/version from competing config rows so authoritative `agent_version` values win regardless of UNION sorting or row order. |
| [`4.1-operation-query-portability`](upstream/4.1-operation-query-portability/status.md), #23693 | X34/X33 | Portable boolean comparisons and text-size expressions, plus the reviewed fix that treats each NULL column as zero instead of discarding the entire row from the estimate. Filtering is preserved; size/pruning decisions for nullable rows intentionally change. Text size is not stored bytes. |
| [`4.1-crossmodel-secret-revisions`](upstream/4.1-crossmodel-secret-revisions/status.md), #23688 | U11 | Any remote unit behind makes a secret reportable; return its latest revision once through indexed offer connections. Tracking uniqueness and upserts are scoped by `(offer_connection_uuid, secret_id, unit_name)`; synthetic-unit cleanup uses one CTE-backed DELETE. |
| [`4.1-nullable-primary-keys`](upstream/4.1-nullable-primary-keys/status.md), #23710 | U15, charm | Require mount container keys and key manifest bases by manifest position, retaining nullable attributes. |
| [`4.1-application-exposure-scope`](upstream/4.1-application-exposure-scope/status.md), #23723 | U15, exposure | Separate application-wide wildcard exposure from endpoint-specific rows and require endpoint IDs in endpoint tables. |
| [`4.1-lookup-key-not-null`](upstream/4.1-lookup-key-not-null/status.md), #23724 | U17 | Explicitly require INT lookup keys and reject absent payload IDs rather than silently treating them as valid zero IDs. |
| [`4.1-lease-pin-conflict`](upstream/4.1-lease-pin-conflict/status.md), [#23729](https://github.com/juju/juju/pull/23729) | X18 | Resolve competing pins with the explicit lease/entity conflict target. Preserve repeated-pin and missing-lease behaviour; reset the test helper's captured results on transaction retries. This is portability, not a current dqlite defect. |
| [`4.1-sequence-upsert-target`](upstream/4.1-sequence-upsert-target/status.md), [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | Target namespace conflicts explicitly, qualify the stored value and share a scalar allocator. At `696c116f79`, a checked upsert and same-transaction lookup replace `76b4a04f09`'s RETURNING implementation to preserve retryable errors. Preserve zero-based, contiguous and namespace-isolated allocation, rollback and import continuity. |
| [`4.1-drop-iif`](upstream/4.1-drop-iif/status.md), [#23751](https://github.com/juju/juju/pull/23751) | X7 | Replace redundant IIF wrappers with equivalent null-check predicates in the connection query and three controller views at `adf9f317a6`. |
| [`4.1-pr-description-guidance`](upstream/4.1-pr-description-guidance/status.md), [#23755](https://github.com/juju/juju/pull/23755) | Agent guidance | Distinguish reviewer context, prospective merge-message content and reproducible reviewer QA from validation history at `15119bb898`. |

Remaining review actions belong in branch records, with coordinator summaries
in the dashboard. GitHub is the source of truth for live review, CI and merge
status.

## Sequence allocation and review follow-ups

See the [branch record](upstream/4.1-sequence-upsert-target/status.md#sequence-allocation-and-review-follow-ups).

## Dbrepl latest-row selection (X16)

See the [branch record](upstream/4.1-dbrepl-latest-row/status.md#dbrepl-latest-row-selection-x16).

## Dbaccessor pool cleanup (U14)

See the [branch record](upstream/4.1-dbaccessor-pool-cleanup/status.md#dbaccessor-pool-cleanup-u14).

## Agent-binary and lease-pin publication evidence

See the [branch record](upstream/4.1-agentbinary-conflict/status.md#agent-binary-and-lease-pin-publication-evidence).

## Nullable primary-key review split

See the [branch record](upstream/4.1-nullable-primary-keys/status.md#nullable-primary-key-review-split).

## Secret-query decisions

See the [branch record](upstream/4.1-crossmodel-secret-revisions/status.md#secret-query-decisions).

## Lookup-key NOT NULL decisions

See the [branch record](upstream/4.1-lookup-key-not-null/status.md#lookup-key-not-null-decisions).

## Objectstore review follow-up

See the [branch record](upstream/4.1-objectstore-conflicts/status.md#objectstore-review-follow-up).
