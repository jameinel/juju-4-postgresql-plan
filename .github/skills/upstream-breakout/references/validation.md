# Upstream breakout validation and dispatch

Supply the upstream base, one named branch/worktree, allowed paths,
expected behavior, prerequisites and required regression cases. Task
selection does not authorize work on other branches.

Reproduce the upstream defect where applicable, then add a regression
alongside the fix. For a behavior-preserving SQL improvement, assert the
existing contract across relevant edge cases. Run focused tests first, then
affected package suites for the assigned paths. Use the target checkout's
standard commands, dependencies and database fixtures. Include actual
dqlite checks where required, and its prescribed race/stress,
watcher-readiness, formatting and lint checks.

Before validation, check `PATH` for the required executables. If needed,
add `$HOME/go/bin` to the front of `PATH` for the validation commands; this
is where the precompiled `txncheck`, `golangci-lint`, `gci` and `stress`
are typically installed. Verify that each needed tool is available and use
the compiled `txncheck`, not `go run`. An absent tool is not a successful
check; report it and do not install dependencies unless required.

For routine code changes, format touched Go files with `gci`, check the diff,
and run repository-configured linters on affected packages rather than every
repository-wide analysis:

```sh
golangci-lint run -c .golangci.yml ./path/to/affected/package/...
txncheck "$PWD"
```

The scoped `golangci-lint` command checks tests but not the whole repo;
`txncheck` walks Go source repository-wide and can catch transaction-test
issues the package linter misses. Neither replaces schema, dependency,
license or vulnerability checks.

`make pre-check` skips static analysis unless invoked as
`STATIC_ANALYSIS=1 make pre-check`. The full run includes other checks and
runs `govulncheck` across `github.com/juju/juju/...` when installed; it can
take substantial time and memory. Do not repeat it for each small, local
follow-up when affected tests, scoped lint, txncheck and PR CI provide the
needed feedback. Run it locally when dependencies or relevant vulnerable call
paths change, when an advisory appears, before release/integration
milestones, and periodically to catch drift that scoped checks and CI might
miss. A skipped or failing check is not a pass.

For schema changes, confirm the release's base-DDL/incremental-upgrade policy.
Use `make rebuild-export` from the branch's own schema; regenerate after
conflicts from the combined schema, not whole-file selection. Exercise
writers -> storage -> export -> legacy import -> consumers, recovery and
supported upgrades. Generation alone is not acceptance; generated output must
be limited to intended changes.

Record the base, final commit, exact commands/build tags and results. State
tests not run and unresolved risks; do not claim broader coverage. Explain the
upstream defect/benefit and any behavior change in the PR, following the
target repository's template. Respect the host's test resource budget and
other owners' state.
