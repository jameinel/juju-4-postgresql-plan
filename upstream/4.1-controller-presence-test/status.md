# Controller presence test synchronization

[Upstream dashboard](../../UPSTREAM-PLAN.md)

## Current state and next actions

- Owner/session: reservation originally assigned to
  `df17884f-b667-42da-a8d1-e4b4f898678d`. The coordinator authorized
  session `52899ed5-0213-4e1e-b5fd-de434c656882` to work on this branch
  and its records on 2026-11-19.
- Branch/worktree: `4.1-controller-presence-test` at
  `.worktrees/4.1-controller-presence-test` in the Juju checkout.
- Upstream 4.1 base observed 2026-11-18: `d380e5e4fba4c9918a0774638ffa3138aeece73e`.
- Local, remote and PR head: `66aef8c256bf5af9e7b29ac67aff0791bca75772`
  (2026-11-19). The follow-up was pushed with approval and verified.
- PR: [juju/juju#23885](https://github.com/juju/juju/pull/23885), published
  with approval against upstream 4.1. GitHub confirmed the head and exact
  approved body on 2026-11-18 at 18:18 UTC.
- Review: Van Cleef requested an explanatory comment and suggested
  extracting shared mock setup. The coordinator agreed to add the comment
  and retain the explicit setup. Published and verified the approved
  `pr.md` body and
  [review reply](https://github.com/juju/juju/pull/23885#issuecomment-7024422314).
- Checks observed 2026-11-19: checks on the new head are queued or in
  progress, with several skipped; human review is required. On the prior
  head executed checks succeeded except Read the Docs. Its linked build
  returned 404 without access, so the failure cause is unknown.
- Next: await review and CI; investigate the earlier documentation failure
  if it recurs and access is available.

## Review follow-up validation (2026-11-19)

At local head `66aef8c256bf5af9e7b29ac67aff0791bca75772`, added the
comment describing the immediate-break window without changing behavior.

- `go test -race -count=1 -timeout=120s
  ./internal/worker/controllerpresence`: passed.
- Race-enabled test binary and `timeout 120s stress -p 4 -count 100
  -timeout 30s -failfast ... -test.run='TestWorker/TestNewConnectionTrackerBroken($|Immediately$)'
  -test.timeout=25s`: 100 runs, zero failures.
- `gci` on the touched file, `git diff --check`, scoped `golangci-lint`,
  and compiled `txncheck "$PWD"`: passed.
- Full local pre-check was not run for this comment-only follow-up;
  GitHub's successful static-analysis check applies to the earlier head.

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
