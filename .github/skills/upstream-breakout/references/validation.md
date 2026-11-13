# Upstream breakout validation and dispatch

Supply the upstream base, one named branch/worktree, allowed paths,
expected behavior, prerequisites and required regression cases. Task
selection does not authorize work on other branches.
Read planning AGENTS.md and keep that contract, revision-specific evidence
and next actions in `upstream/<branch>/status.md`. Shared summaries are
coordinator-owned; do not update them for every validation run.

For SQL/schema work, apply the plan's **Purpose and SQL/schema acceptance**
contract before implementation and again at review:

- Identify the underlying query or stored-type defect and explain how the
  actual SQL/schema change removes it. Check the stored representation and
  normal readers/writers, not only whether Go can bind, scan or round-trip it.
  Do not substitute Go conversions or casts for correcting an incorrectly
  typed schema. Genuine legacy/API conversions and documented driver
  workarounds need their own compatibility or defect rationale.
- If the task's prescribed approach would leave that defect intact, stop and
  ask the user rather than silently implementing it or expanding scope.
- Validate SQL/schema changes on SQLite and real dqlite. Tests on these
  permissive backends do not alone prove that the SQL formulation is portable;
  explain why the offending construct or type mismatch is resolved.
- Keep the final code SQLite/dqlite-only, without PostgreSQL-specific SQL or
  support infrastructure. PostgreSQL is an optional reference for uncertain
  SQL/schema semantics; ask the user when execution is needed. Record any such
  checks separately, and do not claim PostgreSQL acceptance when it was not
  tested. PostgreSQL results do not replace either supported backend's checks.

Reproduce the upstream defect where applicable, then add a regression
alongside the fix. For a behavior-preserving SQL improvement, assert the
existing contract across relevant edge cases. Run focused tests first, then
affected package suites for the assigned paths. Use the target checkout's
standard commands, dependencies, environment and database fixtures, not PoC
environment setup or backend-selection commands. Include actual
dqlite checks where required, and its prescribed race/stress,
watcher-readiness, formatting and lint checks.

For real dqlite, use the target checkout's `make run-go-tests`: it adds the
`dqlite,libsqlite3` tags and configures the bundled `_deps` musl compiler,
headers and static libraries. Do not infer missing dqlite from `pkg-config`
or a `PATH`-only search for `musl-gcc`. Check the paths selected by that
checkout's `scripts/dqlite/Makefile` first, because the target's
`*-install-if-missing` prerequisites can provision dependencies. Reuse the
shared `_deps` described in UPSTREAM-PLAN.md instead of redownloading it.
For example, adjust the package and selection for the change:

```bash
set -o pipefail
timeout 150s make run-go-tests TEST_PACKAGES=./domain/lease/state \
  TEST_ARGS='-v -race -count=1 -timeout 120s' \
  TEST_FILTER='TestStateSuite/TestClaimLeaseSuccessAndLeaseQueries' \
  2>&1 | tee "$LOG_FILE"
```

Omit `TEST_FILTER` to run a package suite. Keep the log outside the
checkout; a bounded command's timeout is not a pass.

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
target repository's template. Use normal tool parallelism defaults, adjusting
for current host load, and respect other owners' state.
