# Controller presence test synchronization

[Upstream dashboard](../../UPSTREAM-PLAN.md)

## Current state and next actions

- Owner/session: `df17884f-b667-42da-a8d1-e4b4f898678d`; reservation
  authorized by the coordinator on 2026-11-18.
- Branch/worktree: `4.1-controller-presence-test` at
  `.worktrees/4.1-controller-presence-test` in the Juju checkout.
- Upstream 4.1 base observed 2026-11-18: `d380e5e4fba4c9918a0774638ffa3138aeece73e`.
- Head: `bca36e3c78e11544af3188b407d9d6ee7f9ea021` (2026-11-18).
  Pushed with approval to `jameinel/juju:4.1-controller-presence-test`;
  remote head verified at the same commit.
- PR: [juju/juju#23885](https://github.com/juju/juju/pull/23885), published
  with approval against upstream 4.1. GitHub confirmed the head and exact
  approved body on 2026-11-18 at 18:18 UTC.
- Blockers: none. Next: await review and CI; checks were queued or pending
  at the observation above.

## Scope and acceptance

Fix the test ordering defect in
`internal/worker/controllerpresence/worker_test.go`. The immediate-break
test can observe the buffered channel receive before the worker records
`connected=false`. Use a realistic closed broken-connection channel and a
worker-action barrier after the state update; preserve immediate-break and
presence-cleanup coverage. Exercise both normal and immediate break tests
with race-enabled stress, the affected suite, formatting and scoped lint.
No production behavior or SQL/schema change is planned.

The high-priority relation-hook task is not dispatchable until its design is
agreed. This independent test-only follow-up has no prerequisite. The
related juju/juju#23873 changes a separate presence issue, not this worker
test. The CI failure and prior non-reproducing stress runs are documented
in the dashboard's controller-presence test-flake section.

## Implementation and validation (2026-11-18)

The original immediate-break test is present in upstream 4.1 at the base
above. Blame attributes it to `e95b6bcc40`, introduced by
juju/juju#21869, merged into 4.0 on 2026-03-04. It is not a test introduced
by an open breakout PR. This independent branch targets upstream 4.1.

The normal test closes the broken channel after observing the connection.
The immediate test closes it as `IsBroken` returns false, and waits for
unit-presence cleanup before asserting that the worker is disconnected.
Both preserve their cancellation-aware receive and cleanup expectations.

Scheduling reproducer: temporarily insert
`time.Sleep(500 * time.Millisecond)` in `worker.go` immediately after
`case <-c.Broken():` and before `w.connected.Store(false)`. With the
original immediate test, `go test -race -count=1 -timeout=30s -run
'TestWorker/TestNewConnectionTrackerBrokenImmediately$'
./internal/worker/controllerpresence` failed with `connected=true`. With
the updated tests and the same instrumentation, `go test -race -count=5
-timeout=30s -run 'TestWorker/TestNewConnectionTrackerBroken($|Immediately$)'
./internal/worker/controllerpresence` passed. This exposes a scheduling
ordering bug, not a Go data race. The temporary delay was removed.

Final uninstrumented validation:

- `go test -race -count=1 -timeout=120s -run
  'TestWorker/TestNewConnectionTrackerBroken($|Immediately$)'
  ./internal/worker/controllerpresence`: passed.
- `go test -race -count=1 -timeout=120s
  ./internal/worker/controllerpresence`: passed before and after the
  temporary instrumentation.
- `go test -c -race -o controllerpresence.test
  ./internal/worker/controllerpresence`, then `timeout 120s stress -p 4
  -count 100 -timeout 30s -failfast ./controllerpresence.test
  -test.run='TestWorker/TestNewConnectionTrackerBroken($|Immediately$)'
  -test.timeout=25s`: 100 runs, zero failures. The binary was removed.
- `gci` on the touched file, `git diff --check`,
  `golangci-lint run -c .golangci.yml ./internal/worker/controllerpresence/...`,
  and `txncheck "$PWD"`: passed.

Full repository pre-check and controller integration QA were not run; this
is a test-only change with no production behavior change.
The user reported running the PR's before/after manual QA on 2026-11-18;
individual output was not independently inspected.
