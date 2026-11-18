# 4.1-install-dependencies

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

2026-11-18 CI investigation: the
[documentation build](https://github.com/juju/juju/actions/runs/44349342389/job/134964325711)
failed importing `drawsvg` 2.5.0 with `NameError: SyncedAnimationConfig`.
The previous passing documentation job installed drawsvg 2.4.2;
the pinned ggarch dependency allows `drawsvg>=2.0`. This is a dependency
change outside this Makefile-only PR. The Read the Docs build also failed,
but its log was inaccessible, so the cause is unconfirmed.
The [unit-test job](https://github.com/juju/juju/actions/runs/44349342389/job/134960478649)
also failed: `controllerpresence/TestNewConnectionTrackerBrokenImmediately`
observed `connected=true` when expecting false. The aggregate test check
failed as well. These failures have not been reproduced locally.

2026-11-18 manual QA: the coordinator confirmed that all documented manual
QA steps passed against local head `57e2ea08c3`. The Snap Store connectivity
issue recovered; the coordinator shut down the QA container. The
coordinator updated `pr.md`, including a snap refresh before installing Go.
The grouped review reply records the completed QA. With coordinator
approval, pushed follow-up commits `f637dd291e` and `57e2ea08c3`; local,
origin and PR head were verified at `57e2ea08c3`. Publication of the
description was separately approved, published and verified against
`pr.md`. The edited review reply was separately approved, published as
[a grouped comment](https://github.com/juju/juju/pull/23838#issuecomment-6985180624)
and verified against `reply-van-cleef.md`. Next: monitor checks and obtain
human review; the three Codex threads remain unresolved. On the pushed head,
the GitHub documentation build and Read the Docs checks failed; their
causes are not yet established. The GitHub Actions run was still in
progress and its failed-job log was not yet available. Other checks were
pending, passed or skipped.

2026-11-18 scope clarification: the coordinator decided Go must be
installed before Make runs. The dependency target must not install Go or
enforce its version; rely on Go's `go.mod` toolchain selection instead.
QA now installs the Go snap before invoking Make. The coordinator's initial
fresh-container run reached the Go snap install but failed with a snapd EOF;
the daemon log showed shutdown while the request was in flight. This is
not evidence of a Makefile defect, and a complete run has not yet been
repeated for the revised behavior. Local head is `57e2ea08c3`, two commits
ahead of the published `5c6eb270e6`.

The revised target was checked with Go absent, yq snap failure, yq missing
from PATH after a simulated install, incompatible yq, and real yq success.
ShellCheck on the expanded recipe and `git diff --check` passed.
`make pre-check` passed with static analysis skipped. The earlier Go
version-branch probe applied to an intermediate implementation and does not
validate the final version-free implementation. The full privileged
install was not repeated. The PR description and review reply were updated
locally; neither the push nor publication has been approved.

2026-11-18 review follow-up: coordinator authorized takeover by session
`07dde829-85a3-4c53-9fd0-e784ccd783db`. PR
[23838](https://github.com/juju/juju/pull/23838) and origin head remain
`5c6eb270e6`; local head is `f637dd291e` on
`4.1-install-dependencies`. The local worktree is clean and one commit
ahead. No push or review publication has been approved.

The coordinator reviewed all points in the
[Van Cleef review](https://github.com/juju/juju/pull/23838#pullrequestreview-5934455988):
retain the effective GOBIN lookup and existing tool-version policy; remove
the installer recipe tests from static analysis rather than adding the
suggested edge cases; fix yq installation failure diagnostics; and document
the expected golangci-lint output format. The missing-yq test suggestion is
moot after removing the tests. The three prior Codex threads have replies
and remain unresolved.

On `f637dd291e`, shell stubs verified failed snap installation, successful
snap installation without yq on PATH and incompatible yq; an existing real
yq passed the capability probe. The restricted PATH used for the stubs
also emitted Go-not-found warnings while Make evaluated unrelated variables.
The static-analysis shell suite, ShellCheck on the expanded recipes, shfmt
on the changed shell script, and `git diff --check` passed. `make pre-check`
passed but skipped static analysis. Full static analysis and a fresh
privileged installation were not repeated. The removed test no longer
provides automated coverage of yq and golangci-lint compatibility.

The reviewed PR description is prepared in `pr.md` and the grouped response
in `reply-van-cleef.md`. On the published head, GitHub checks passed or
were skipped; human review is still required. Next: review the local commit
and artifacts with the coordinator, and obtain separate approval for each
push or publication action.

2026-11-16 review follow-up: coordinator authorized takeover by session
`a7cb8548-f5f6-43de-a460-86cf53db2f6e`. Local, origin and PR head are
`5c6eb270e6`. The new Codex
[GOPATH finding](https://github.com/juju/juju/pull/23838#discussion_r4575422384)
did not reproduce with Go 1.27.3 or 1.27.4: `go env GOBIN` returned the
effective first GOPATH entry's bin directory, including spaces, with
GOBIN and GOPATH unexported and GOPATH configured through `go env -w`.
The coordinator requested a shared lookup rather than a spaces-specific
regression. Commit `5c6eb270e6` uses `go env GOBIN` for both the
installer and static analysis, removing the fallback and duplicate logic.

The existing installer compatibility test passed with Go 1.27.3.
ShellCheck passed for the expanded installer recipe; `git diff --check`
and `make pre-check` passed, but the latter skipped static analysis.
An isolated Go configuration showed both consumers use a saved GOBIN.
Full static analysis and a fresh privileged install were not repeated.
The existing `pr.md` still matches the live description and needs no
change. With separate approval, pushed the follow-up and posted and
verified the [reply](https://github.com/juju/juju/pull/23838#discussion_r4590452385),
which matches `reply-codex-gopath.md`. The three Codex threads remain
unresolved.

The live PR requires human review. On the prior head, the documentation
job was cancelled after 24 hours and `Check Tests Passed` failed; the
cause of the cancellation is not established. Other checks passed or
were skipped. Checks on the new head are queued or in progress, with
some already passed or skipped. Next: monitor CI and obtain human
review.

2026-11-14 review follow-up: owner transferred by coordinator to session
`6d5559ea-28f8-4389-aa51-de3e98e84db4`. Local, origin and PR head are
`0ed7ee32e4` on `4.1-install-dependencies`. Both Codex findings are addressed:
the selected yq is probed for JSON input/output syntax, and the selected
golangci-lint must satisfy the static-analysis 2.14.x rule. Existing tools
are not overwritten. The coordinator approved both changes.

`cd tests && PATH="$HOME/go/bin:$PATH" ./main.sh static_analysis
run_install_dependencies_compatibility` passed, including incompatible
versions, PATH shadowing, unsupported yq syntax, and nonzero probe exits.
ShellCheck passed for the test scripts and expanded Make recipes; shfmt
and `git diff --check` passed. `make pre-check` passed but skipped static
analysis. Full static analysis and a fresh privileged install were not
repeated for this follow-up. Running the real Go-tools target installed
the previously absent `govulncheck` in `/home/jameinel/go/bin`; it did not
replace existing Go tools.

After explicit approval, pushed the follow-up and posted and verified the
[yq reply](https://github.com/juju/juju/pull/23838#discussion_r4575398069).
After separate approval, updated and verified the PR description and posted
and verified the
[golangci-lint reply](https://github.com/juju/juju/pull/23838#discussion_r4575400794).
The saved `pr.md` and both reply files match the published text. Neither
thread was resolved. Checks on the new head were queued or in progress at
the last observation, with some skipped; human review is still required.

Next: monitor CI and obtain human review.

## State before review

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
