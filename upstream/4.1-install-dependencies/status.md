# 4.1-install-dependencies

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Owner/session: `ddb7e2df-6760-4f1f-94e5-8bcfa0df4332`.
Reserved with coordinator on 2026-11-11.
Branch: `4.1-install-dependencies`.
Worktree: `.worktrees/4.1-install-dependencies` (shared `_deps` symlink).
Base: `bd970f693c8e5060973016fe9059108570fa5bd3`, verified against
upstream `4.1` on 2026-11-11.
Head: `d8625d3f045c64f3ffcc667a0c22a14619a4d019`.
PR: https://github.com/juju/juju/pull/23838, open against `juju/juju:4.1`;
head and description verified on 2026-11-13.
GitHub rechecked 2026-11-13 20:51 UTC: the PR remains open at that head,
with `REVIEW_REQUIRED` and aggregate checks `SUCCESS`. This aggregate does
not establish that every check ran or replace the recorded installation
evidence and its limits.
Blockers: remaining review gate and PR functional QA.
Next: obtain review, monitor checks and perform the PR's functional QA. Obtain
separate approval before publishing replies or making further PR changes.

Coordinator clarification on 2026-11-11: the task must audit and install
missing developer dependencies, not merely verify them. Version checks are
also useful. The earlier verification-only decision was superseded.
`tests/main.sh` requires `curl`, `yq` and `shellcheck`; static analysis
also uses `golangci-lint` 2.14, `shfmt`, `go-licenses` and `govulncheck`.
The existing target uses sudo for snap and apt. `go install` would write
to the user's GOBIN and may overwrite existing binaries, while a Make
recipe cannot permanently change the caller's PATH. CI pins golangci-lint
2.14.0 and shfmt 3.7.0 but uses latest for two other tools; it does not
specify standalone gci or stress versions.
Local inspection found the named tools on PATH and golangci-lint 2.14.0.
No local version mismatch was established. `tests/README.md` also names
`expect`, but the runner does not check it and no Expect scripts were
found; do not make it a prerequisite without evidence of a consumer.

2026-11-11 progress: added `verify-development-tools` to `Makefile` and
called it after the existing snap and SQLite dependency targets. It checks
the caller's PATH, reports installation commands, and validates the yq
syntax and golangci-lint version without installing developer tools.
`make verify-development-tools` passed using the actual local binaries.
Two `install-dependencies` invocations passed with the existing privileged
snap and apt prerequisites explicitly skipped (`make -o`); this is not a
clean installation test. A reduced PATH reported missing tools as expected.
The targeted static shell analysis passed shellcheck and shfmt.
`make pre-check` passed but skipped static analysis. Full static analysis
and clean privileged installation have not been run. The coordinator
decided full static analysis is unnecessary for this tool-verification
breakout. ShellCheck passed on the expanded verification recipe.

2026-11-11 clarification and new validation: the coordinator confirmed
that installation, not verification alone, is the goal. The revised patch
installs missing tools and keeps version checks. A fresh Ubuntu 24.04 LXD
container, `juju-install-deps-ddb7e2df`, mounts the source read-only; only
`make` was bootstrapped before testing. The initial full target failed
because `go.mod` specifies `1.27.3` but the snap track is `1.27`. The patch
selects the major/minor snap track. The second attempt installed the snaps
and apt packages but failed because the mount caused `/home/ubuntu` to be
root-owned before cloud-init. Its ownership was corrected in the container.
A subsequent full run installed the Go tools successfully.

Final validation on 2026-11-11:

- In a second fresh Ubuntu 24.04 LXD container, bootstrap only `make`, mount
  the checkout read-only after cloud-init, and run `make install-dependencies`
  with `GOFLAGS=-p=2`: passed. A repeat passed and SHA-256 checks confirmed
  the six Go-tool binaries were unchanged. The clean run predates only the
  final Go-version comparison correction; the full target was rerun after it.
- The Go 1.9 and 1.27rc1 checks rejected versions below 1.27.3. Equal,
  1.100.0 and absent-Go dry runs selected the expected branches and the
  absent-Go case selected `1.27/stable`.
- The container verified tool versions, compiled and linked a SQLite C
  program, and confirmed the static-analysis command receives both the
  default Go binary directory and an explicitly configured GOBIN.
- ShellCheck passed for the expanded installation and verification recipes;
  `git diff --check` passed. Full static analysis was not run by coordinator
  decision. Both disposable containers were removed after validation.
- The first clean run emitted existing Makefile evaluation warnings that Go
  was missing before installing it; installation still completed.

2026-11-12 follow-up: the coordinator removed `verify-development-tools`
and requested a top-level Go tool list. Commit `2937c3d9f5` preserves that
removal and derives executable names from the listed module paths. In a
fresh Ubuntu LXD container bootstrapped with `make` and Go, running
`make install-development-go-tools` installed all six tools. A second run
preserved their SHA-256 hashes. ShellCheck and `git diff --check` passed.
This follow-up did not rerun the complete privileged dependency target or
full static analysis. The disposable container was removed.

2026-11-13 publication: included the coordinator's Makefile documentation
help alignment change in commit `d8625d3f04`. `git diff --check` and
`make -n docs` passed for that change. Pushed the branch and created the PR
with the reviewed description. The dependency installer is unchanged from
`2937c3d9f5`; the full privileged target and static analysis were not
rerun for this whitespace-only follow-up.

Scope and acceptance: independently inventory required and skipped
development tools, add appropriate installation/verification to
`make install-dependencies`, and validate clean and repeated installs plus
the affected checks. Preserve the existing privilege requirements.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-FIXES.md`:

`4.1-install-dependencies` | U18 | not proposed |  | not implemented

### Upstream register snapshot

U18 | tooling | `Makefile` `install-dependencies`; static-analysis prerequisites | `install-dependencies` covers Go snap and `libsqlite3-dev`, but not `shellcheck`, `gci`, `golangci-lint` or `stress`. Without `shellcheck`, `STATIC_ANALYSIS=1 make pre-check` stops at dependency checks; without SQLite headers it fails schema generation with `sqlite3.h` missing. The run also reports skipped `go-licenses`, `shfmt` and `govulncheck` checks when those tools are absent. | Found 2026-11-03; not implemented. Independently inventory required and skipped development tools, add appropriate installation/verification to `make install-dependencies`, and validate clean and repeated installs plus the affected checks. See `4.1-install-dependencies` in UPSTREAM-PLAN.md.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### `4.1-install-dependencies` — developer tooling

On a separate branch, update `make install-dependencies` to install or verify
the tools needed for the repository's normal build, formatting, tests and
static analysis: `shellcheck`, `gci`, `golangci-lint`, `stress`, and
`libsqlite3-dev`. The current target already installs `libsqlite3-dev`, but
running `STATIC_ANALYSIS=1 make pre-check` without it fails compiling
`go-sqlite3` (`sqlite3.h` missing); without `shellcheck`, the test harness
stops before linting. Static analysis also reports `shfmt` missing (shell
analysis disabled), `go-licenses` missing (licence analysis skipped), and
`govulncheck` missing (vulnerability analysis disabled).
Inventory other required or skipped tools and include them where appropriate,
using the versions and installation conventions supported by the target
branch. Ensure installed executables are on the PATH used by checks.
Do not silently require new privileges or mix this tooling change into the
operation query review. Validate on a clean environment and a repeated
invocation, then run the checks that previously failed or were skipped.

## U18 provenance snapshot (2026-11-11 migration)

Found 2026-11-03; not implemented. Independently inventory required and skipped development tools, add appropriate installation/verification to `make install-dependencies`, and validate clean and repeated installs plus the affected checks. See `4.1-install-dependencies` in UPSTREAM-PLAN.md.
