# 4.1-install-dependencies

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

Found 2026-11-03; not implemented. Independently inventory required and skipped development tools, add appropriate installation/verification to `make install-dependencies`, and validate clean and repeated installs plus the affected checks. See `4.1-install-dependencies` in UPSTREAM-PLAN.md.

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
