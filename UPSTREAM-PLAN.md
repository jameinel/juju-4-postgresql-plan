# Stock upstream Juju 4.1 remaining work

Coordinator dashboard and reservation index for stock-upstream work. Read
[AGENTS.md](AGENTS.md), the selected `upstream/<branch>/status.md` and the
target checkout's guidance before acting. Use that checkout's build/test
commands, fixtures and environment; PoC backend selection and operating
instructions do not apply. Branch records own execution details; summaries
here are dated snapshots, not live GitHub state or permission to take over
another owner's worktree.

Summary baseline: existing records migrated on 2026-11-11, including
the 2026-11-09 review snapshot and subsequent recorded updates. This
reorganization did not refresh GitHub, CI, QA or branch ownership.

[PoC plan](PLAN.md) | [Finding register](UPSTREAM-FIXES.md) |
[Completed summary](UPSTREAM-DONE.md) |
[Earlier dashboard snapshot](history/upstream-plan-before-branch-records-2026-11-11.md)

## Purpose and SQL/schema acceptance

These breakouts bring SQL and schema improvements identified by the PostgreSQL
PoC into stock upstream 4.1. The deliverable is correct, explicit SQL and
appropriately typed schemas running on SQLite and dqlite, not PostgreSQL
support. Do not introduce PostgreSQL-specific SQL, backend switching, runtime
dependencies or a PostgreSQL test-harness requirement into these patches.
Use portable SQL where it expresses the intended behavior on both supported
backends; PostgreSQL can help expose ambiguity, but does not define Juju's
domain semantics.

Fix the SQL/schema root cause, not just the Go conversion boundary. A
successful bind, scan or round trip does not establish that the stored type,
query semantics or SQL formulation is correct. Do not preserve an incorrectly
typed column by adding Go formatting/parsing, or hide a type mismatch with a
cast solely to make the current representation work. For example, U21 stores
integer relation IDs as integers; legacy text-key conversion belongs in
import, not in every normal read and write.

Boundary conversions remain appropriate for genuine legacy import/export or
external API compatibility. Documented driver-defect workarounds, such as the
temporary nullable-BOOLEAN casts in the branch records, have a separate rationale and removal
condition; they are not precedent for masking schema defects. This goal does
not turn explicitly scoped tooling, decoding or driver fixes into SQL rewrites.

Before implementing, check that the task's prescribed approach addresses the
underlying SQL/schema issue. PoC implementations are evidence, not mandatory
designs. If a task says to preserve a representation or avoid schema changes
but that appears to leave the root problem intact, stop and ask the user.
Do not silently follow a conflicting prescription or expand the breakout.
Schema changes still require the release-policy and compatibility checks
in the validation guide; this goal does not waive them.

Acceptance requires explaining how the actual SQL/schema change removes the
identified issue, as well as validating behavior on SQLite and real dqlite.
Passing those tests alone does not demonstrate portable SQL: both can accept
the same permissive construct. PostgreSQL execution is optional and available
to resolve uncertain SQL/schema semantics; ask the user when it is needed.
Report it separately and never claim PostgreSQL acceptance or execution from
SQLite/dqlite results.

## Working agreement

- Refresh and record the upstream 4.1 base (`B`). Check current code and
  existing reviews for completed work before implementing anything.
- Start each independent branch from `B`; continue open reviews on their
  existing branches. Do not duplicate landed work or reuse its branch for
  unrelated follow-ups. The charm/exposure and #23756/#23759 stacks are
  documented exceptions.
- Dedicated worktrees are allowed. Assign one branch/worktree per owner;
  never switch, reset or rebase another owner's checkout. A worktree's
  presence does not imply permission to modify it.
- Place new worktrees under the primary checkout's `.worktrees/`. Share
  existing dependencies with a `_deps` symlink to `../../_deps` from a
  direct child worktree; do not redownload them unnecessarily.
- `go` is a snap on this host. Never redirect its output, or the output
  of `make` targets that run it, straight to a file: AppArmor silently
  discards it, and the exit code is unaffected. Pipe instead, for example
  `set -o pipefail; go test ... 2>&1 | tee "$LOG"`.
- Keep one review concern per branch. Multiple commits are fine when they
  serve that concern. Shared generated files are an integration conflict,
  not automatically a reason to stack independent reviews.
- Implement using the APIs, schema layout, helpers and test infrastructure
  present in the target branch. Missing helpers require a small local
  solution or clarification, not dependencies on another project.
- Stop for unresolved behaviour, unexpected dependencies or ownership
  conflicts. Do not weaken tests to make a patch pass.

For dispatch and validation details, read the
[breakout validation guide](.github/skills/upstream-breakout/references/validation.md)
when starting or reviewing implementation work.

## Breakout claims

Reservations are serialized by the coordinator. Re-read this index and
the branch record before reserving; the claim is not permission to alter
another owner's worktree. Record the owner and record link here before
starting. Keep paused/blocked work claimed. Release or transfer only with
the owner's or user's authorization. Base, worktree, head, progress and
next actions belong in the branch record, not this table.

| Task record | Owner/session |
|---|---|
| [`4.1-boolean-and-aggregate-literals`](upstream/4.1-boolean-and-aggregate-literals/status.md) | `24713b22-6d3f-4622-9c56-7756485516d1` |
| [`4.1-cmr-ingress-order`](upstream/4.1-cmr-ingress-order/status.md) | `87b89739-8e7d-4e6e-a852-8cf8fa0e4d61` |
| [`4.1-controllerupgrader-stream-result`](upstream/4.1-controllerupgrader-stream-result/status.md) | `533e88e8-ff4b-44c8-82d0-d01fac7ff80b` |
| [`4.1-install-dependencies`](upstream/4.1-install-dependencies/status.md) | `ddb7e2df-6760-4f1f-94e5-8bcfa0df4332` |
| [`4.1-ssh-host-key-conflict`](upstream/4.1-ssh-host-key-conflict/status.md) | `d36531ca-29ec-454e-af43-9c0f627b8c93` |
| [`4.1-storage-filesystem-attachment-row`](upstream/4.1-storage-filesystem-attachment-row/status.md) | `fc0b395c-35d9-4277-ba1d-3bc4779db286` |
| [`4.1-unitstate-relation-key`](upstream/4.1-unitstate-relation-key/status.md) | `750b409e-35d5-4196-9b64-56ee36a49f56` |
| [`4.1-x16-grouped-columns`](upstream/4.1-x16-grouped-columns/status.md) | `48eb9888-f082-47eb-a43f-75ac280c4a3e` |

Absence from this table does not establish that an existing branch is
unowned. Historical records may omit assignments; confirm before takeover.

## Branch summaries

The coordinator refreshes this snapshot at handoff, landing or a status
sweep. Workers update their branch record, not this dashboard on each
commit or review. GitHub is authoritative for live PR state. A merged PR
does not establish functional QA or PoC integration.

| Branch/task record | Recorded summary |
|---|---|
| [`4.1-agentbinary-conflict`](upstream/4.1-agentbinary-conflict/status.md) | Landed upstream; retain recorded QA and PoC reconciliation limitations. |
| [`4.1-application-exposure-scope`](upstream/4.1-application-exposure-scope/status.md) | Depends on charm; rebase/regenerate after charm lands. |
| [`4.1-binary-key-columns`](upstream/4.1-binary-key-columns/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-blockcommand-order`](upstream/4.1-blockcommand-order/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-boolean-and-aggregate-literals`](upstream/4.1-boolean-and-aggregate-literals/status.md) | Pending independent SQL portability breakout. |
| [`4.1-changestream-latest-row`](upstream/4.1-changestream-latest-row/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-cmr-ingress-order`](upstream/4.1-cmr-ingress-order/status.md) | Confirm upstream ordering contract; review, CI triage and functional QA. |
| [`4.1-controller-export-null`](upstream/4.1-controller-export-null/status.md) | Independent backup NULL-fidelity review, CI and functional QA. |
| [`4.1-controller-node-id-text`](upstream/4.1-controller-node-id-text/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-controllerupgrader-stream-result`](upstream/4.1-controllerupgrader-stream-result/status.md) | Review, CI and live QA; reconcile after landing. |
| [`4.1-crossmodel-secret-revisions`](upstream/4.1-crossmodel-secret-revisions/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-cte-membership-subqueries`](upstream/4.1-cte-membership-subqueries/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-dbaccessor-pool-cleanup`](upstream/4.1-dbaccessor-pool-cleanup/status.md) | Landed upstream; retain recorded QA and PoC reconciliation limitations. |
| [`4.1-dbrepl-latest-row`](upstream/4.1-dbrepl-latest-row/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-dbrepl-output`](upstream/4.1-dbrepl-output/status.md) | Landed upstream; retain recorded QA and PoC reconciliation limitations. |
| [`4.1-dqlite-nullable-boolean`](upstream/4.1-dqlite-nullable-boolean/status.md) | Refresh after merged keymanager fix; temporary casts have a dependency-fix removal condition. |
| [`4.1-drop-iif`](upstream/4.1-drop-iif/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-install-dependencies`](upstream/4.1-install-dependencies/status.md) | Pending developer-tooling task; validate clean and repeated installation. |
| [`4.1-into-4.2`](upstream/4.1-into-4.2/status.md) | Landed upstream; retain recorded QA and PoC reconciliation limitations. |
| [`4.1-keymanager-insert-id`](upstream/4.1-keymanager-insert-id/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-lease-migration-tests`](upstream/4.1-lease-migration-tests/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-lease-pin-conflict`](upstream/4.1-lease-pin-conflict/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-local-secret-revisions`](upstream/4.1-local-secret-revisions/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-lookup-key-not-null`](upstream/4.1-lookup-key-not-null/status.md) | Independent INT-key guard; reconcile the combined guard when changes meet. |
| [`4.1-lxd-default-bases`](upstream/4.1-lxd-default-bases/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-model-config-precedence`](upstream/4.1-model-config-precedence/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-nullable-primary-keys`](upstream/4.1-nullable-primary-keys/status.md) | Land charm before exposure; retain the explicit stack exception. |
| [`4.1-objectstore-conflicts`](upstream/4.1-objectstore-conflicts/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-operation-query-portability`](upstream/4.1-operation-query-portability/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-postgresql-tests`](upstream/4.1-postgresql-tests/status.md) | Infrastructure-only; scope confirmed, backend design still proposed. |
| [`4.1-pr-description-guidance`](upstream/4.1-pr-description-guidance/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-removal-force-boolean`](upstream/4.1-removal-force-boolean/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-secret-latest-revision`](upstream/4.1-secret-latest-revision/status.md) | Pending; coordinate with the remote-secret review before dispatch. |
| [`4.1-sequence-upsert-target`](upstream/4.1-sequence-upsert-target/status.md) | Existing review; continue the recorded acceptance contract and outstanding actions. |
| [`4.1-sql-standard-syntax`](upstream/4.1-sql-standard-syntax/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-sql-test-literals`](upstream/4.1-sql-test-literals/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-sqlite-integers`](upstream/4.1-sqlite-integers/status.md) | Landed upstream; see the record and finding register for QA and PoC reconciliation. |
| [`4.1-ssh-host-key-conflict`](upstream/4.1-ssh-host-key-conflict/status.md) | Review, CI and live QA, including the Kubernetes unit path. |
| [`4.1-storage-filesystem-attachment-row`](upstream/4.1-storage-filesystem-attachment-row/status.md) | Merge hold: attachment invariant, duplicate rejection and migration QA need a decision. |
| [`4.1-unitstate-relation-key`](upstream/4.1-unitstate-relation-key/status.md) | Review/CI and mainline propagation; reconcile the different PoC workaround after landing. |
| [`4.1-x16-grouped-columns`](upstream/4.1-x16-grouped-columns/status.md) | Review/CI, no-CA QA and multi-certificate integration coverage. |
| [`dqlite-returning-errors`](upstream/dqlite-returning-errors/status.md) | Unclaimed dependency follow-up; select release/backport before implementation. |
| [`lease-migration-fixture-timestamps`](upstream/lease-migration-fixture-timestamps/status.md) | Unclaimed, low-priority test-only follow-up. |

## D55 implementation review (2026-11-06)

The root-cause review was a design assessment, not fresh test, CI or QA
acceptance. Preserve the storage merge hold above. U21's integer-schema
rework and grouped-column certificate preservation supersede the earlier
boundary-only/syntax-only approaches. Branch records retain the details;
the [merged assessment](UPSTREAM-DONE.md#d55-merged-implementation-assessment-2026-11-06)
explains why completed work should not be reopened merely for conversions.

## Nullable primary-key review split

The nullable primary-key stack is charm #23710 -> exposure #23723. Exposure owns
the combined primary-key guard; lookup #23724 owns an independent INT-key
guard. When exposure and lookup meet, regenerate the combined schema,
remove the global guard's INT exemption (retain only single-column INTEGER
rowid keys), and run both guards. Do not tighten it before lookup keys land
or add the other concern to an independent PR. Both U15 fixes are required
for PoC bootstrap reintegration; charm alone is insufficient.

## Mainline integration requests

The `4.1-into-4.2` integration [#23775](https://github.com/juju/juju/pull/23775)
merged into `main` on 2026-11-10; implementation, review and validation
history are in [UPSTREAM-DONE.md](upstream/4.1-into-4.2/status.md#41-to-main-integration).
It included `4.1` through `bd36cef535`, not subsequent commits. At the
2026-11-10 snapshot `4.1` was at `79e9dbc31e`; assess remaining
merge-forward work against refreshed branch heads, not the landed proposal.
Live controller and migration QA for the integration were not recorded.

At `main` head `0483b81440`, four SQL `IIF` expressions remain in
`domain/model/state/controller/state.go` and
`domain/schema/controller/sql/0007-model.sql`. The cleanup in
[#23751](https://github.com/juju/juju/pull/23751) still targets `4.1` and
was not part of #23775. Prepare a separately scoped mainline cleanup
preserving behavior, or verify its inclusion in a later merge-forward.
Check current heads and obtain approval before publishing or merging.
The separate automated merge-forward proposal [#23709](https://github.com/juju/juju/pull/23709)
is contextual, not a user-owned review task.

## Candidates needing bounded scope before dispatch

These register findings are not ready-made assignments. Identify remaining
production paths, behaviour contracts and focused tests before branching;
split conflicts by operation and preserve retries, idempotency and errors.

| Finding | Candidate scope / next action |
|---|---|
| X7 | Published as #23751; follow the linked branch record. |
| X24 | Published as #23750; follow the linked branch record. Do not duplicate. |
| X3/X4, other upserts | Sequence allocation is implemented in #23731. Inventory any other remaining operations before dispatch; do not recreate that fix or combine unrelated conflict paths. |
| U11 watcher follow-up | Per-offer-connection remote watcher scoping is separate from #23688's offered-application scope. Agree the event/isolation contract and tests before dispatch. |
| X33 stored-bytes follow-up | A stored-bytes metric such as `pg_column_size` needs backend-aware state code; #23693's text-size work does not implement it. Scope separately, not as a mechanical rewrite. |

D55 audit additions (2026-11-06). Each construct below remained in upstream
`4.1` at `148e09f015` and is masked in the PoC by a Stage 4 override,
generator rewrite or PoC-only commit (DIALECT.md section 4). The shared fix
must be valid SQLite and contain no PostgreSQL SQL. Base-DDL changes alter
the schema hash, so the 4.1 beta release policy applies.

References to removing PoC overrides below are reconciliation obligations
for the PoC integration owner after the shared fix lands, not instructions
to edit PostgreSQL paths in a stock-upstream breakout.

| Finding | Candidate scope / next action |
|---|---|
| X29 predicate-free joins | Five schema views: `v_model_all`, `v_secret_permission`, `v_secret_owner`, `v_machine_target_agent_version`, `v_unit_target_agent_version`. Use explicit `CROSS JOIN` (precedent X38, #23695). Coordinate `v_model_all` with #23751. Delete the matching override hunks afterwards. |
| X16 `v_user_last_login` | Remove the bare column from `controller/sql/0019-model-last-login.sql` while keeping a `time.Time`-scannable result on dqlite. Verify the last-login read paths on SQLite and dqlite. |
| Duplicate FK name | Rename one `fk_offer_connection_uuid` on `application_remote_consumer` (`0034-cross-model-relation.sql`). Name-only change; check that nothing refers to the name. |
| X31 constraint order | `v_application_constraint` orders by implicit `rowid`, and `GetApplicationConstraints` documents insertion order, so treat order as a contract. Add an explicit sequence column to `constraint_tag`, `constraint_space` and `constraint_zone`, written from the input slice index. Every writer (`application/state/application.go`, `machine/state/placement.go`, `model/state/model/state.go`) deletes and reinserts the whole list, so the index is enough; `network/state/space_remove.go` deletes single rows, which only leaves gaps. Order the view by the new column. Check that export and import preserve the order. Removes D17's PostgreSQL-only identity columns. |
| X7 remaining functions | `IFNULL` (4 files), `GROUP_CONCAT`, `INSERT OR IGNORE`, `DATETIME('now')` in removal and upgrade state. Port the non-lease parts of PoC `8930466847` per package. Separate from #23751 (`IIF`). |
| D16 lease time | Decided: keep database time. `lease/state` claim/extend use `datetime('now', $duration)` and `leaseexpiry` compares `expiry < datetime('now')`. Port `CURRENT_TIMESTAMP` for the comparison and compute the expiry from the database time read in the transaction (PoC 05-lease-db-time), keeping whole-second text. Do not port the PoC's earlier controller-clock binding. A clock that never runs faster than real time across leader changes, clock steps and outages (like the old Juju global clock) is a separate design for both backends. |
| Lease migration `date('now')` | `domain/lease/state/migration.go` selects leadership leases with `expiry >= date('now')`: midnight UTC today, so it also exports leases that expired earlier today. PostgreSQL accepts it only as a function-style cast. Confirm intent (likely `expiry > CURRENT_TIMESTAMP`, or all leases) and add a test with an expired lease. |
| X8 bare `user` | Quote `user` in seven state files and 15 schema DDL references. Valid SQLite; removes the generator's quoting. |
| X5 scalar `MAX` | `SetLastLogTransferTime` (`modelmigration/state/model/state.go`); use `CASE` as in PoC `1e25d5bf39`. |
| X2 empty `IN` | sqlair renders an empty slice as `IN ()`. Choose portable rendering in sqlair or caller guards; inventory callers that can pass empty slices. Removes the Stage 3 driver rewrite. |
| `BOOLEAN DEFAULT 0` | `flag.value` (`controller/sql/0016-flag.sql`); use `FALSE`. Audit seed rows that write `0` / `1` to `BOOLEAN` columns. |
| `STRFTIME` defaults (tentative, D56) | Seven `STRFTIME('%Y-%m-%d %H:%M:%f', 'NOW', 'utc')` defaults in `0003-changelog.sql`, `0001-changelog.sql`, `0015-charm.sql`, `0018-machine.sql` and `0028-cleanup.sql`. Possible change to `CURRENT_TIMESTAMP`, which is portable, always UTC, and matches the whole-second text that `DATETIME('now', 'utc')` already writes. Not committed: it drops millisecond precision on SQLite. No reader needs that precision today, but keeping it costs little (DIALECT.md section 4). Decide before proposing. |
| Charm `create_time` order | `GetLatestPendingCharmhubCharm` (`application/state/charm.go`) takes the first row of `ORDER BY c.create_time DESC`, which depends on the default because `setCharmState` omits `create_time`. Charms created at the same instant order arbitrarily, and import binds Go-formatted times into the same text column. Bind the time from the domain clock in UTC and add a deterministic tiebreak, or order by an explicit key. Independent of the `STRFTIME` decision. |
| X25 `allocate_public_ip` | Add to the #883 removal checklist in the nullable-Boolean branch record: when Juju bundles the dqlite fix, make the column `BOOLEAN` and its export type `*bool`, removing the Stage 4 override. |

## PoC reconciliation handoff

Stock-upstream workers record source/merge revisions and integration
obligations in their branch records; they do not restack the PoC or
maintain PoC stage progress. The coordinator updates [PLAN.md](PLAN.md)
when dependencies or priorities change. The integration owner follows
[UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md) and records verified
reconciliation in [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md). Preserve U17's
combined-guard obligation, U11's independent reconciliation, and D43's
charm/exposure comparison against the eventual landed result.

The PostgreSQL regression branch and deferred stream-shutdown work are
[separate infrastructure work](upstream/4.1-postgresql-tests/status.md),
not requirements for individual upstream SQL/schema breakouts. Dbrepl
remains outside the PoC scope.

## Dispatch

Use the [upstream-breakout skill](.github/skills/upstream-breakout/SKILL.md)
for selection and reservations, the branch record for its task contract,
and the [validation guide](.github/skills/upstream-breakout/references/validation.md)
for stock-upstream checks. Daily summaries synchronize the dashboards;
branch workers maintain only their assigned records and review artifacts.
