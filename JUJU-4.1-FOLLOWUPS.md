# Juju 4.1 follow-ups

These are general Juju 4.1 improvements, independent of the PostgreSQL PoC
and its upstream breakout plans. They are not assigned or scheduled.

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
