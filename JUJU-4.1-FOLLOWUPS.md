# Juju 4.1 follow-ups

These are general Juju 4.1 improvements, independent of the PostgreSQL PoC
and its upstream breakout plans. They are not assigned or scheduled.

## Investigate the cross-model relation egress watcher test timeout

**Status:** Unassigned, not implemented. Found 2026-11-13.

The Jenkins `make race-test` run for
[juju/juju#23766](https://github.com/juju/juju/pull/23766) failed in
`domain/crossmodelrelation`, specifically
`TestWatcherSuite/TestWatchRelationEgressNetworks`. The failure was
`timed out waiting for idle state: after setup 5`, not a Go race-detector
report. The subsequent `go2xunit` conversion also failed with
`orphan end test`, leaving no test report files.

The affected test deletes two egress networks in separate transactions during
the same harness case. In the Jenkins trace, the deletions produced separate
change-stream terms: event 29 was dispatched, while event 30 started dispatch
but did not complete before the idle timeout. A plausible interleaving is
that the watcher blocks sending the first notification while the harness
waits for the stream to become idle before consuming it; the second dispatch
then cannot complete. This has not been reproduced or conclusively proven.

The test and relevant watcher infrastructure are unchanged from the PR's
4.1 merge base, and the same two deletions are present on upstream 4.1. The
PR changes storage behavior, not this watcher test. Investigate the
synchronization and confirm the cause. Combining the deletions in one
transaction may eliminate the problematic interleaving and match the watcher
test harness's single-transaction convention; verify the intended final
fallback notification and use focused race and stress tests.

## Fix the machine SSH handler test teardown race

**Status:** Unassigned, not implemented. Found 2026-11-13.

The [unit test job](https://github.com/juju/juju/actions/runs/43090138216/job/130764797950)
for [juju/juju#23805](https://github.com/juju/juju/pull/23805) reported a
data race in `internal/worker/sshserver/handlers/machine`. An SFTP handler
goroutine called `handleError`, logging through `WrapCheckLog`, while the Go
test runner was finishing a test. The report does not identify which SFTP
subtest left the handler running.

The test helper `startSSHTestServer` closes its listener during cleanup but
does not wait for handler goroutines to finish. Investigate the complete
server and session lifecycle and ensure test cleanup waits for all relevant
handlers before the test and its logger finish. Preserve the SSH and SFTP
behavior under test; do not merely suppress the race report.

The affected handler, test helper, logger and dependencies were unchanged
from the PR's parent. Local `go test -race -count=20
./internal/worker/sshserver/handlers/machine` and 100 repeated SFTP subtest
runs passed; the race is intermittent and was not reproduced locally.
Validate the fix with focused race tests and stress testing of the affected
goroutine lifecycle.
