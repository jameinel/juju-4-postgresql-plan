# Juju 4.1: lost relation-changed after an A -> B -> A settings change

## Summary

A relation-changed hook can read remote relation data that differs from
the version the uniter records for it. If the data then returns to its
earlier content (A -> B -> A), the recorded and current versions match
and no further hook runs. The charm keeps acting on B until an unrelated
change happens.

The cause is two properties together:

1. The version recorded for a hook is chosen when the hook is queued,
   but the data is fetched later, on the hook's first `relation-get`.
2. In 4.x, versions are hashes of the content, so A always has the same
   version.

3.6 has the first property but its versions only ever increase, so the
same timing produces an extra hook rather than a missed one. This is a
4.x regression, independent of the database backend.

**Evidence status:**

- Code paths inspected at 4.1 `bd970f693c` (2026-11-12 UTC) and
  confirmed unchanged at 4.1 tip `1a0d0c78bc` (2026-11-17 UTC). 3.6
  compared at `e23e56080e`.
- Observed once, on a development build. Not yet reproduced on stock
  4.1; the exact ordering in that incident is unproven.

## How we found it

We ran a cross-model relation test on a development build based on
4.1-beta3.1, using a default dqlite controller. `juju-qa-dummy-source`
sets `token=foobar` in its relation-joined hook. Two
`juju-qa-dummy-sink` units run `relation-get token` in relation-changed
and report "Token is foobar" or "Waiting for token".

Both sinks were healthy. After a controller restart, we suspended and
resumed the relation. On resume, within the same second:

- The source re-ran relation-joined and set `token=foobar`.
- Both sinks ran relation-joined then relation-changed, and **both read
  an empty token**.
- sink/0 got another relation-changed, read `foobar` and recovered.
- sink/1 got no further hook. It stayed waiting for over an hour,
  including through another agent restart.

Afterwards, the source's data contained `token=foobar`. Both sinks had
recorded the version of that current data (`4136104582745174885`, from
hash `e3d84ef9…`). Neither had a pending or failed hook. A fresh
`relation-get` on sink/1 returned `foobar`. Juju had recorded that sink/1
was up to date, while the charm's last read had found no token.

The same test passed on a PostgreSQL controller and across controllers
in both directions. Full timeline and database queries:
[D54 follow-up](../../stages/09-bootstrap.md#sourcelog-follow-up-fixed-hook-versions-and-an-aba-timing-window).

## How hooks read relation data today

The relation cache code is the same in 3.6 and 4.1, so both releases
give charms the same view of relation data:

| What the hook reads | Behavior |
|---|---|
| `relation-list` members | Membership delivered through joined/departed hooks; fixed for the hook. |
| Data of the unit that triggered the hook | Cache cleared when the hook starts [context]; fetched live on first read, then fixed for the hook [cache]. |
| Other units' data, already read in an earlier hook | Cached value, kept until that unit's own hook runs. The charm doesn't "see the future" for units it has already read. |
| Other units' data, never read | Fetched live on first read, then fixed. |
| Application data | Cleared whenever a context is built [prune]; live on first read in every hook. |
| Anything, after an agent restart | Cache is in memory only; live on first read. |

Each bag is fixed once read, so a hook never sees one bag change. There
is no common snapshot across bags, though, so different bags can come
from different moments.

## The flaw

| Step | What the code does |
|---|---|
| Queue | The resolver copies the watcher's version into the hook [select]. |
| Read | The first `relation-get` fetches current data; the request carries no version [read] [settings]. |
| Finish | The queued version is recorded, not the version of what was read [commit] [ack]. |
| Compare | Versions are a hash of the sorted settings [hash] [version]; another hook runs only if current != recorded [compare]. |

That allows:

| Step | Sink uniter | Source data |
|---|---|---|
| 1 | Queues relation-changed with version(A). | A |
| 2 | Hook starts, hasn't read yet. | Changes to B |
| 3 | First `relation-get` reads B (no token). | B |
| 4 | Hook still running. | Back to A |
| 5 | Hook finishes; records version(A). | A |
| 6 | Recorded A == current A; no further hook. | A |

No event is lost and no hash collides. The data without the token
hashes to `2e538147…`, which is distinct from `e3d84ef9…`.

**Fit to the incident:** during resume, the source's data plausibly
held addresses but no token (B) until its joined hook set the token
again (A). sink/1 matches the sequence above. sink/0 also read B, but
had queued version(B), so recorded B != A and it got another hook.

**Not proven:** the logs don't show the queued versions or sub-second
ordering, and the original controllers are gone.

### Why 3.6 didn't lose updates

In 3.6 the settings version increases on every write [v36] [v36-inc]. A
hook queued at version n that reads n+1 records n, and the current
version is at least n+1, so another hook runs. The worst case was a
redundant hook.

## Impact

Charms rely on Juju to notify them when relation data differs from what
they last saw. Juju may skip intermediate values but must eventually
deliver the current one. Here the charm keeps stale data, which may be
credentials or endpoints. Agent restarts don't recover it. It is hard
to diagnose: `relation-get`, `juju exec` and the database all show
correct current data, and the agent is idle.

## Recommended fix: record the version that was read

When the hook reads the data of the unit or application that triggered
it, record that data's version instead of the queued one. With
content-hash versions this is the right comparison: if current data
equals what the charm read, nothing is owed, and A -> B -> A is valid
coalescing as long as the hook read A.

1. **The controller returns the version with the data.** Read the
   settings hash in the same transaction as the settings
   [settings], and add a version to `params.SettingsResult` [params].
   The agent should not compute the hash itself: any format mismatch
   would make the versions never match and queue relation-changed
   forever.
2. **The cache keeps settings and version together.**
3. **Substitute before the hook is marked Done.** `Execute` saves the
   hook info with step `Done` [done], and `Commit` records whatever
   that contains. Update `ChangeVersion` before that state is saved.
4. **Only for the triggering unit or application.** If unit/0's hook
   reads unit/1's data, unit/1's recorded version is unchanged, so its
   own hook still runs: possibly redundant, never missed.
5. **If the hook didn't read the data, keep the queued version.** The
   charm was notified and chose not to read.

Restarts behave correctly [steps]:

| Agent restarts | Saved step | Result |
|---|---|---|
| While the hook runs | `Pending` | Treated as a hook error; a retry reruns the hook, reads again and records that. |
| After the hook, before commit | `Done` | Commit only, using the substituted version. |

`juju resolved --no-retry` commits the queued version without rerunning
the hook. That is its contract, and it stays unchanged.

Compatibility: an older controller returns no version, so the agent
keeps today's behavior. An older agent ignores the new field. A facade
version bump may be required by convention.

This keeps today's read behavior, including different bags coming from
different moments, and needs no schema change.

## Alternatives

**Versions that always increase (3.6 behavior).** Add a per-bag
revision that increases on every write, and use it as the watcher
version. This also prevents the missed update, without API changes.
Costs: a schema change and an upgrade step for existing rows, and the
revision must not restart when a unit leaves and rejoins (a model-wide
sequence avoids that). It produces more redundant hooks than the
recommended fix.

**A fixed snapshot per hook.** At hook start, read all remote data the
hook may need, with versions, in one transaction. Serve every read in
the hook from that snapshot and record its versions. This removes
differences between bags as well as the missed update. It is a larger
design: which data to include, whether to keep the "don't see the
future" caching across hooks, what a retry uses, payload size, and
whether credential-bearing data may be stored on the agent. No
database transaction should stay open while the hook runs.

Rejected:

- Hashing on the agent: risks endless relation-changed hooks if the
  formats differ.
- Recording the version current at commit: it describes data the hook
  may not have seen.
- Forcing a hook after every resume: other A -> B -> A paths remain.

## Testing

Write a deterministic uniter test, using barriers rather than sleeps:

1. Queue relation-changed at version(A) and block the hook before its
   first read.
2. Change the data to B, then let the read happen.
3. Restore A and wait until the uniter's watcher has seen it.
4. Let the hook finish; check the recorded version and the resolver's
   next operation.

On 4.1 today: the hook reads B, records A, and no further hook runs.
With the recommended fix: B is recorded and another hook reads A.
Testing the resolver alone isn't enough; the test has to check what the
hook actually read.

Also cover:

- application data;
- the relation-changed that follows relation-joined;
- a hook that doesn't read the data;
- a restart at `Pending` and at `Done`;
- unit/1's data read during unit/0's hook;
- an older controller that returns no version.

Run with `-race` and bounded stress. A live reproduction is possible
with a gated test charm, but it is timing-sensitive and not required.

## Diagnosing affected units

Suspect a unit that is idle and acting on stale data while current
relation data and its recorded version both look current. Equal
versions don't show what an earlier hook read, and a fresh
`relation-get` can't reconstruct it. Compare what the charm recorded or
applied with the source's write and restart times before intervening.
Any new logging should record hook and version identities, not
settings, which may contain credentials.

[select]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/relation/resolver.go#L188-L206
[context]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/runner/context/contextfactory.go#L238-L260
[cache]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/runner/context/cache.go#L70-L95
[prune]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/runner/context/cache.go#L49-L58
[read]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/api/agent/uniter/relationunit.go#L144-L173
[settings]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/domain/relation/state/relation.go#L2296-L2326
[params]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/rpc/params/internal.go#L263-L266
[commit]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/operation/runhook.go#L284-L291
[done]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/operation/runhook.go#L195-L200
[ack]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/relation/state.go#L103-L135
[steps]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/resolver.go#L166-L197
[hash]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/domain/relation/state/relation.go#L2475-L2490
[version]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/domain/relation/state/relation.go#L2121-L2134
[compare]: https://github.com/juju/juju/blob/1a0d0c78bcea8ab077735a7f404b2fb31ac93702/internal/worker/uniter/relation/resolver.go#L337-L368
[v36]: https://github.com/juju/juju/blob/e23e56080e27d9f537b872d87f6f41af72ba8625/state/settings.go#L31-L33
[v36-inc]: https://github.com/juju/juju/blob/e23e56080e27d9f537b872d87f6f41af72ba8625/state/settings.go#L436-L455
