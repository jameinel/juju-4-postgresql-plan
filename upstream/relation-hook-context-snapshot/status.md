# relation-hook-context-snapshot

[Shared direction](../../AGENTS.md) |
[Upstream dashboard](../../UPSTREAM-PLAN.md) |
[Finding register](../../UPSTREAM-FIXES.md)

## Current state and next actions

Recorded 2026-11-12 UTC, continuing the local 2026-11-11 D54 investigation.
**High-priority transactional correctness concern; unclaimed.** The user
confirmed the required charm contract below. The exact cause of the
observed incident remains unproven; this is not a confirmed wild incident
or a dqlite-specific diagnosis.

This is an independent Juju 4.1 correctness issue, not a SQL/schema
breakout or a dependency of the other upstream/proof-of-concept work.
The [standalone team discussion draft](discussion.md) contains the
stock-source analysis, proposed reproduction approaches and implementation
options without requiring that planning context.

The user has since destroyed the original PoC controllers. Retained logs
and database snapshots are the incident evidence; new live experiments
require separate approval.

- Owner/session: unassigned; recording this task does not claim implementation.
- Branch/worktree: none created. This directory is a stable task identifier.
- Upstream source reviewed: `4.1` at
  `bd970f693c8e5060973016fe9059108570fa5bd3`, verified against the GitHub
  branch tip on 2026-11-12 UTC. Relevant selection/cache/read/acknowledgment
  and hash-version paths are present in stock upstream.
- Runtime evidence: development build at
  `f3340c67107829347ee750f7e40e806e563411e9`; distinct from stock-source
  inspection. No clean stock runtime reproduction has been run.
- Phase: behavior contract and discussion draft recorded; source-path
  verification complete. Implementation design and deterministic
  reproduction pending. No fix, regression execution or upstream issue/PR.
- Dashboard registration: left for coordinator handoff because
  `UPSTREAM-PLAN.md` has other sessions' uncommitted reservation changes.
  The finding register and D54 record link to this task.

Next actions:

1. Reserve with the coordinator and refresh stock upstream 4.1 before
   implementation. The paths were inspected at the revision above;
   check for subsequent changes and existing implementations/reviews before
   branching. Runtime reproduction remains separate from source inspection.
2. Select a design that couples the delivered context snapshot with its
   trigger identity and persisted acknowledgment. Specify restart/retry
   behavior and compatibility before implementation.
3. Implement a deterministic regression for the contract and event
   orderings below. Do not depend on reproducing the original live race.
4. Prepare one upstream correctness breakout, including diagnostic guidance
   for affected units. Obtain separate approval for implementation scope,
   live experiments and publication; preserve the retained D54 evidence.

## Required transactional contract

Charms trust relation-hook notifications to tell them when previously
delivered data has changed. They should not need to poll or revisit
unchanged-looking data defensively.

**Record "hook fired with context X", where X is the immutable context
snapshot made available at the start of that hook. Do not acknowledge the
controller's latest context Y at completion, or associate X's version with
different content fetched lazily during execution.**

The trigger/snapshot boundary must supply coherent content and identity
before execution. Once selected for a running hook, its remote relation
settings cannot silently advance. A late first read, repeated read,
different key or whole-databag read must not expose newer content than
that assigned snapshot.

Coalescing is allowed: not every intermediate value needs a hook. However,
if current content differs from the last delivered context, Juju must
eventually attempt another hook delivering the changed context. Seeing an
equal version or hash is not sufficient if it describes content other
than what the hook was actually given. Preserve existing pending-hook and
failure/retry semantics; a failed hook is not a successful acknowledgment.

Here A means a snapshot with `token=foobar`; B means one without the token.

| Delivered context | Concurrent/current content | Required behavior |
|---|---|---|
| A | A -> B -> A | The hook reads A throughout. B may be skipped; no additional hook is required merely for that intermediate state. |
| B | B -> A | The hook reads B throughout, and Juju eventually attempts a hook delivering A. An earlier delivery of A does not satisfy this obligation. |
| A | A -> B, remaining B | The hook reads A throughout, and Juju eventually attempts a hook delivering B. |
| Trigger labeled A, but hook reads B | B -> A | Invalid delivery: trigger identity, available content and acknowledgment disagree. Do not treat final A as proof that the hook saw A. |

This is a charm transactional guarantee, not merely a workload-status
display issue or an optimization to avoid redundant hooks.

## Evidence and limits

The original dqlite consumer initially had two healthy units.
On resume at 2026-11-12 00:38:10 UTC, both first changed hooks read no
token. Unit 0 received another changed hook and read `foobar`; unit 1
remained waiting/idle with an empty last-hook token file.

Later read-only inspection found `token=foobar` in the current source
databag and both units acknowledging its current hash-derived version.
The source rewrote the same token during resumed joined hooks in the
same second. Unit workers had restarted shortly before resume.

The retained INFO logs do not identify each selected hook version,
snapshot contents or subsecond CMR transitions. Therefore they cannot
prove the proposed ABA ordering. Full timeline, selected database values,
source revision and read-only comparison queries are in
[D54's source/log follow-up](../../stages/09-bootstrap.md#sourcelog-follow-up-fixed-hook-versions-and-an-aba-timing-window).

At that PoC revision, source inspection found:

- `internal/worker/uniter/relation/resolver.go:188-205` selects a
  post-joined changed hook's version from a watcher snapshot.
- `internal/worker/uniter/runner/context/contextfactory.go:238-260`
  invalidates the remote unit cache, but
  `internal/worker/uniter/runner/context/cache.go:72-95` fetches settings
  lazily. `api/agent/uniter/relationunit.go:144-173` passes no selected
  version to `ReadRemoteSettings`.
- `internal/worker/uniter/operation/runhook.go:287-291,350-353` commits
  the original `hook.Info` and does not change it on remote updates.
  `internal/worker/uniter/relation/state.go:103-135` records that version.
  Completion-time replacement with the latest version was not found.
- `domain/relation/state/relation.go:2475-2490,2125-2134` derives versions
  from content hashes. A -> B -> A can reuse A's version, including across
  membership reentry. `resolver.go:340-362` under the uniter relation
  directory compares only version inequality for subsequent changed hooks.
- The inspected scope-deletion, reentry and remote-replacement state paths
  update settings and their hash transactionally. No split database write
  was established; separate trigger selection and settings reads can still
  observe different committed states.

A candidate failure is selection at A, a fresh read of intermediate B,
restoration to A, then acknowledgment of the original A. The resolver can
see equal versions and suppress recovery even though the charm saw B.
This does not require a hash collision or a lost watcher notification.

## Scope and design constraints

Fix the correspondence between a relation-hook trigger, the remote data
available to that hook and its acknowledgment. Cover unit and application
databag paths, guaranteed post-joined changed hooks and normal updates.
Define the snapshot scope for other remote units/relations exposed through
the context during design review; do not silently mix snapshot-bound reads
with live reads while claiming a transactional context.

Do not freeze unrelated hook-tool APIs or disable the unit's own legitimate
relation-setting writes. Keep local writable data distinct from immutable
remote snapshots. Preserve relation lifecycle, authorization and unit/model
isolation.

An increasing revision or incarnation token may help identify deliveries,
but is not sufficient by itself: it must identify the content actually
available to the hook. Do not substitute version-counter changes, forced
extra hooks, polling or status repair for coherent snapshot delivery.
Choose the capture/storage/API boundary after inspecting stock upstream;
this record does not prescribe a PostgreSQL implementation or dependency.

## Acceptance criteria

1. Capture a coherent snapshot and identity before hook execution; persist
   acknowledgment of that delivery, never of whatever is current at
   completion. Coalescing before execution must select a new coherent
   content/identity pair, not retain an old label with new content.
2. Hold an A hook before its first read, change current data to B, and
   assert all its remote reads still return A. Repeat with a key not read
   until after the change, repeated reads and whole-databag reads.
3. Exercise A -> B -> A while an A hook is running. It must not read B;
   the test must allow B to be skipped without demanding an unnecessary
   additional A hook.
4. Deliver B, then restore A, including where A was delivered earlier.
   Require an eventual A hook and actual A reads, not merely a watcher
   event, acknowledged version or idle agent. Also cover A -> B that
   remains B, and coalesced updates whose final content differs.
5. Cover changes between selection and context creation, during execution,
   and around acknowledgment. Include post-joined changed hooks, unit/app
   settings, scope reentry and independent consumers.
6. Exercise pending/running/completed operation restart and retry paths.
   Retain the delivery snapshot or explicitly establish a new coherent
   delivery under reviewed retry semantics; never silently reload current
   content under the old identity. Check persisted state round trips and
   old-agent/controller compatibility if their representation changes.
7. Preserve existing join/change/depart/broken ordering, legitimate local
   writes and error reporting. Errors acquiring a coherent snapshot must
   not produce a success-shaped empty context or acknowledgment.
8. Use `tc` and the existing suites; run affected concurrent code with
   `-race` and bounded stress. Validate SQLite and real dqlite if persistence
   changes. PostgreSQL execution and new live deployments are not acceptance
   prerequisites; no validation has been run for this proposed breakout.

## Recognizing confused units in the wild

A possible symptom is an idle unit retaining stale applied data or waiting
for a value that the current remote databag contains. Another consumer can
be healthy while both persisted versions appear current. Equal hashes
alone do not prove that either unit was given the corresponding content.

For read-only triage, correlate the last relevant hook and its recorded
output/applied state, source writes and scope/restart timing, current
authorized relation data, and each unit's persisted delivery state.
Distinguish current API or `juju exec` reads from the historical hook
context; a new read does not reconstruct what that hook saw. Do not label
every stale unit an ABA incident without this distinction and evidence.

The breakout should document diagnostic correlation of hook/delivery IDs,
snapshot identities and acknowledgment boundaries without dumping
credential-bearing databags. Specify what existing logs cannot establish
and review any additional instrumentation separately. Do not silently
repair a suspected unit by replaying hooks, changing tokens, cycling its
relation or restarting agents; preserve evidence and obtain operational
approval first.
