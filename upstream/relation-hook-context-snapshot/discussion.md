# Juju 4.1: relation-hook context and acknowledgment can diverge

## Summary

There is a source-level race in Juju 4.1 that appears capable of leaving a
charm with stale relation data while the uniter considers it up to date.
The hook's change version is selected from a watcher snapshot, but its
remote settings are fetched separately, lazily, without that version.
The hook can therefore read content other than the content identified by
its trigger and subsequent acknowledgment.

An A -> B -> A transition makes the consequence particularly clear:
the charm can read B while the uniter records delivery of A, then suppress
another hook because current content is A again.

This is a Juju 4.1 relation-delivery correctness issue, not a database
portability task. It does not depend on another development branch or a
different database backend. Cross-model relations and controller restarts
were involved in the motivating observation, but should not be assumed
necessary to expose the underlying race.

**Evidence status:** the relevant paths have been inspected in stock
upstream 4.1 at `bd970f693c8e5060973016fe9059108570fa5bd3`, verified as
the upstream branch tip on 2026-11-12 UTC. A deterministic failure
reproduction on an unmodified stock build has not yet been run. The exact
event ordering in the motivating incident remains unproven.

## The guarantee at risk

Charms rely on hook notifications rather than polling relation data.
Juju need not deliver every intermediate state, but it must attempt
another update when current content differs from the last context
delivered to the charm.

The required invariant is:

**A hook's trigger, immutable remote-data context and acknowledgment must
describe the same delivery. Record "hook fired with context X", not
"context was Y when the hook completed".**

Once that context is assigned, a late first read must not obtain newer
content. Repeated reads, different keys and whole-databag reads must remain
consistent with the assigned snapshot. Legitimate writes to the unit's own
relation data are a separate concern.

Here A is a databag containing `token=foobar`; B is the same bag without
the token. B is not necessarily an entirely empty databag.

| What the hook actually receives | Current-state transitions | Required behavior |
|---|---|---|
| A | A -> B -> A | The hook keeps reading A. B may be skipped; an extra A hook is not required merely because B existed. |
| B | B -> A | Attempt a later hook delivering A, even if an earlier hook had already received A. |
| A | A -> B, remaining B | Attempt a later hook delivering B. |
| B, but recorded as A | B -> A | Invalid: equality between recorded A and current A hides a real difference from the charm's last delivered context. |

Content-hash reuse is therefore not inherently wrong. Comparing content
hashes can implement valid coalescing, provided the recorded hash
identifies the snapshot actually delivered. Replacing hashes with an
increasing counter alone would not establish that correspondence.

## Motivating observation

On a 4.1-beta3.1-based development build using dqlite, a source application
and two consumers initially exchanged `foobar` successfully. Following
cross-model relation suspension/resumption:

- Both consumers' first resumed changed hooks read no token.
- One consumer received another changed hook and read `foobar`.
- The other remained waiting/idle, with an empty last-hook token file.
- Current relation data contained `foobar`, and both consumers' persisted
  relation state acknowledged the current source settings version.
- A subsequent unit-worker restart did not cause a recovery changed hook
  on the affected consumer.

The source issued `relation-set token=foobar` from resumed joined hooks
in the same second as the consumers' empty reads. Existing INFO logs do
not record the selected versions or establish subsecond ordering across
the source, controller and consumers.

This is an observed delivery/read mismatch, but it does not prove that
the specific ABA interleaving below caused that incident. The runtime
observation was not made on an unmodified stock build; the corresponding
code paths were checked independently in stock 4.1.

## What the stock 4.1 code does

The following links are pinned to the inspected upstream revision.

1. [Hook selection][select] takes `ChangeVersion` from the remote watcher
   snapshot. This also happens for the guaranteed relation-changed hook
   after relation-joined.
2. [Hook-context construction][context] invalidates the triggering remote
   unit's cached settings. It does not capture those settings alongside
   the selected version.
3. [The relation cache][cache] fetches a remote databag on its first read.
   This gives a stable cached bag after that read, but not a snapshot of
   the content available when the hook was selected or started.
4. [The API read][read] sends relation/local-unit/remote-unit identity to
   `ReadRemoteSettings`, without the selected change version or snapshot
   identity. [State retrieval][settings] reads current committed settings.
5. [Hook commit][commit] passes the original `hook.Info` to the completion
   callback, and [relation-state update][ack] records its `ChangeVersion`.
   The running operation's [remote-update callback][remote-updates] is
   empty. The inspected path does **not** substitute a newer version at
   completion.
6. [Settings hashing][hash] and [version conversion][version] allow
   identical restored content to have the same version as before.
   [The resolver][compare] schedules another changed hook only when
   remote and recorded versions differ.

The [resolver loop][loop] refreshes the watcher snapshot after operations.
That does not repair a mismatch if the recorded version and the final
watcher version are both A.

This is not evidence that a database transaction exposes partially written
settings. Separate, individually consistent transactions can still select
version A and later read content B.

### A plausible failure ordering

| Step | Uniter/hook | Current source content |
|---|---|---|
| 1 | Select a changed hook with version A. Its remote-unit cache is invalidated but not populated. | A |
| 2 | Keep the hook before its first remote read. | Changes to B |
| 3 | First `relation-get token` fetches B and returns no token; the charm applies B. | B |
| 4 | Let watcher state return to A while the hook is still running. | Restored to A |
| 5 | Successful completion records the originally selected version A. | A |
| 6 | Resolver compares recorded A with current A and finds no further operation. | A |

The charm's last input was B, but the uniter acts as though it delivered A.
No hash collision or lost watcher notification is necessary.

## Plausible reproduction approaches

These are proposed experiments, not completed reproductions.

### 1. Deterministic uniter/context regression

Use explicit barriers in the existing resolver, operation and context test
fixtures, or a composed harness spanning them:

1. Start with a source bag A and select a changed hook carrying A's version.
2. Prepare its context with a cold remote-unit cache, and block before
   the hook's first remote read.
3. Make the backing settings read return B; release the read and record
   the value actually delivered to the hook.
4. Before permitting completion, restore A and explicitly deliver the
   corresponding watcher state. Synchronize on watcher processing, not
   just the database write.
5. Complete the hook, inspect persisted acknowledgment and ask the resolver
   for its next operation.

The suspected current result is: actual read B, acknowledgment A, and no
next changed operation. A resolver-only test is insufficient: it could
prove that equal versions suppress a hook without establishing which
content the context exposed.

With a fix, either the assigned A context must read A, or pre-execution
coalescing must explicitly assign a coherent B context/version and later
deliver A. It must not execute with an A identity while exposing B.
Use barriers rather than sleeps, then run race detection and bounded stress.

### 2. Gated minimal charms on a disposable stock 4.1 model

Use a provider that publishes a configurable token and a consumer with a
minimal shell relation-changed hook. Place them on separate machines so
the consumer's machine-wide hook lock does not block the provider.

The consumer needs two out-of-band gates: one immediately before its first
`relation-get`, and another after recording that read but before exit.
Put markers/gates in its local filesystem, not in relation data.

1. Establish a healthy baseline with a third value such as `initial`.
2. Publish A (`foobar`) to trigger the consumer's changed hook. Wait for
   its pre-read marker.
3. Remove the token to publish B. Confirm the committed databag through
   the controller API without executing a command through the blocked
   consumer's uniter.
4. Release the first gate. Confirm the consumer recorded the absent token
   and is waiting at the pre-exit gate.
5. Restore exactly A. Avoid timestamp/counter keys or other bag changes
   that would prevent the full settings hash from returning to A.
6. Confirm the consumer's watcher has processed restored A, then release
   the exit gate.
7. Check actual applied data, current relation data, persisted delivery
   state and subsequent hooks. An idle agent alone is not success.

Release gates through SSH or another out-of-band test mechanism, **not
`juju exec` on the blocked consumer**: that command may wait behind the
hook being held.

Step 6 is important. A controller API read showing A does not prove that
the consumer watcher has processed A. Without a harness or scoped
instrumentation confirming that boundary, this charm-based experiment
remains timing-sensitive. It is a useful candidate, not a promise of an
immediate live reproduction.

### 3. Lifecycle-assisted reproduction

Use an ordinary dqlite controller, separate offer/consume models, one source
and two consumers. Verify both have applied A, then suspend/resume the
cross-model relation while the source republishes A from its joined hook.
Record each consumer independently.

Run a separate variant with a controller reconnect/restart; do not assume
it is required. Reentry can naturally introduce address-only settings
before the source restores the token, but the required ordering may be
rare. Bound repetitions, stop on a mismatch and retain evidence. Do not
use failure to reproduce as evidence that the contract is safe.

## Potential implementation plan

The design should make a hook delivery an explicit combination of content,
identity and lifecycle state. The following is a discussion proposal, not
an implementation decision.

### Preferred starting point: materialize the context before execution

1. **Define the delivery boundary.** Treat a watcher event as a reason to
   consider a hook, not as proof of the payload that a later read will
   return. Before execution, select a coherent settings/identity pair.
   If newer data is coalesced at preparation time, update the selected
   delivery consistently before running the hook.
2. **Read content and identity together.** Provide a domain/state operation
   that obtains the relevant remote settings, versions and membership in
   one bounded read transaction, including archived settings that departure
   hooks are entitled to read. Expose a typed snapshot through the agent
   API. Keep snapshot semantics in the domain and orchestration/auth in the
   facade; do not combine independent reads and call them atomic.
3. **Give the hook a private immutable view.** Seed its remote-data cache
   from that snapshot. Late first reads and reads of other keys/bags in
   the defined scope must not fall through to current live data. Avoid
   mutable cache sharing with actions, commands or later hooks. Keep the
   unit's own writable settings separate.
4. **Persist the delivery consistently.** Integrate snapshot identity and
   any necessary replay data with the pending/running/completed operation
   lifecycle. Commit the identity of the context actually assigned, not
   a fresh completion-time lookup. Capturing other units' bags must not
   silently acknowledge their separate joined/changed notifications.
5. **Reconcile against that delivered context.** After completion, evaluate
   current watcher state against the acknowledged snapshot. Final content
   different from the delivered context must remain pending; identical
   final content may legitimately coalesce.
6. **Specify recovery and compatibility.** Decide whether a restart/retry
   retains the original snapshot or explicitly begins a new coherent
   delivery. Never attach live content to an old identity silently.
   Review state serialization, rolling agent/controller upgrades,
   authorization and snapshot retention before choosing the wire/storage
   representation.

Do not hold a database transaction open for the duration of a charm hook.
Capture a bounded snapshot and release the transaction before execution.
Persisting relation payloads also needs a deliberate data-retention and
access-control decision: databags can contain credentials.

### Alternatives and tradeoffs

| Approach | Benefit | Main concern |
|---|---|---|
| Eager, coherent per-hook snapshot | Straightforward immutable reads; potentially no historical settings store | Payload/read/memory cost for large contexts; restart persistence and snapshot scope need design. |
| Server-side snapshot handle with versioned reads | Lazy transfer while preserving a selected historical view | Requires retained snapshot/history, cleanup, authorization and recovery; substantially more machinery. |
| Return a version with each existing lazy read and acknowledge that version | Can improve correspondence for a bag that was read | Does not by itself provide a start-of-hook snapshot or prevent different bags/late reads from seeing different times. |
| Replace hashes with monotonic revisions only | Distinguishes A/B/A occurrences and may cause a recovery hook | Does not stop a hook labeled A from reading B; not a complete transactional-context fix. |

Start by evaluating the eager snapshot approach against actual context size
and lifecycle requirements. A narrower change is acceptable only if it
establishes the agreed invariant, not merely if it produces an extra hook
in the motivating example.

## Questions for team discussion

- What is the precise snapshot scope: triggering unit/app bags, all remote
  data in the relation, or all remote relation data available to the hook?
  How should relation membership/listing align with those frozen reads?
- At which boundary may a queued notification be coalesced into a newer
  coherent delivery, and how is that reflected in pending operation state?
- What should retry/restart preserve, and can it explicitly select a new
  context without breaking existing hook-ordering and recovery guarantees?
- Can existing hash-based identity be retained once delivery is coherent,
  or is a separate revision/incarnation identity useful? Neither choice
  should substitute for snapshotting.
- How should older agents and controllers negotiate the behavior, and what
  evidence is sufficient to support a 4.1 fix/backport?

## Acceptance and field diagnosis

Cover both normal and guaranteed post-joined changed hooks, unit and
application bags, late first reads, repeated/different-key reads,
independent consumers, scope reentry and pending-operation recovery.
Explicitly test both valid A/B/A coalescing and the obligation to deliver A
after a B context. Exercise updates around capture, execution and commit.
Preserve departure/archived-settings reads and join/change/depart/broken
ordering. Test the actual payload and persisted acknowledgment, not only
events.

For field diagnosis, an idle unit retaining stale applied data while
current relation data and recorded versions agree is suspicious, but not
proof of ABA. Correlate actual hook output/applied state, source writes,
scope/restart timing and persisted delivery state before intervention.
Current API or `juju exec` reads do not reconstruct an earlier hook context.

If additional instrumentation is needed, correlate hook/delivery IDs,
snapshot identities and acknowledgment boundaries without dumping
credential-bearing data. Existing INFO logs may be insufficient to
establish the exact ordering.

The impact is potentially more serious than an incorrect status message:
a charm may retain stale credentials, endpoints or other configuration
while trusting that no further notification is owed. Recovery may require
an unrelated future change because persisted state already appears current.
The fix should restore the delivery guarantee rather than require charms
to poll or operators to replay hooks.

[select]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/relation/resolver.go#L188-L206
[context]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/runner/context/contextfactory.go#L238-L260
[cache]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/runner/context/cache.go#L70-L124
[read]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/api/agent/uniter/relationunit.go#L144-L173
[settings]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/domain/relation/state/relation.go#L2296-L2326
[commit]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/operation/runhook.go#L284-L291
[ack]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/relation/state.go#L103-L135
[remote-updates]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/operation/runhook.go#L350-L353
[hash]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/domain/relation/state/relation.go#L2475-L2490
[version]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/domain/relation/state/relation.go#L2121-L2134
[compare]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/relation/resolver.go#L337-L368
[loop]: https://github.com/juju/juju/blob/bd970f693c8e5060973016fe9059108570fa5bd3/internal/worker/uniter/resolver/loop.go#L95-L157
