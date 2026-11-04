# Stock upstream Juju 4.1 remaining work

Use this standalone queue for a stock upstream 4.1 checkout. Read the shared
rules, assigned task and target repository's AGENTS.md/contribution guidance;
no PoC checkout, configuration or test setup is required.

This is the recorded 2026-10-30 status snapshot. Check upstream code, GitHub
reviews and CI before acting; implementation, landing and PoC reintegration
are different states. Keep this document focused on pending actions.
[UPSTREAM-DONE.md](UPSTREAM-DONE.md) records implemented work and rationale;
[UPSTREAM-FIXES.md](UPSTREAM-FIXES.md) owns finding/provenance/status mappings.
Neither is prerequisite reading for an assigned task.

## Working agreement

- Refresh and record the upstream 4.1 base (`B`). Check current code and
  existing reviews for completed work before implementing anything.
- Start each independent branch from `B`; continue open reviews on their
  existing branches. Do not duplicate landed work or reuse its branch for
  unrelated follow-ups. The charm/exposure stack below is the only exception.
- Dedicated worktrees are allowed. Assign one branch/worktree per owner;
  never switch, reset or rebase another owner's checkout. A worktree's
  presence does not imply permission to modify it.
- Share existing dependencies with a `_deps` symlink to `../juju/_deps`
  when creating a worktree; do not redownload them unnecessarily.
- Keep one review concern per branch. Multiple commits are fine when they
  serve that concern. Shared generated files are an integration conflict,
  not automatically a reason to stack independent reviews.
- Implement using the APIs, schema layout, helpers and test infrastructure
  present in the target branch. Missing helpers require a small local
  solution or clarification, not dependencies on another project.
- Stop for unresolved behaviour, unexpected dependencies or ownership
  conflicts. Do not weaken tests to make a patch pass.

### Validation without repeated full static analysis

For routine code changes, run the affected tests (with `-race` and stress
when concurrency requires them), format touched Go files with `gci`,
check the diff, and run the repository-configured linters on affected
packages rather than invoking every repository-wide analysis:

```sh
golangci-lint run -c .golangci.yml ./path/to/affected/package/...
go run ./scripts/txncheck/main.go "$PWD"
```

The scoped `golangci-lint` command checks tests but not the whole repo;
the separate `txncheck` walks Go source repository-wide and is cheap
(about 0.6 seconds and 64 MB RSS with a warm cache in the lookup-key
checkout). It can catch transaction-test issues the package linter
misses. Neither command replaces schema, dependency, licence or
vulnerability checks. Check that the binaries are on `PATH`; an absent
tool is not a successful lint run.

`make pre-check` skips static analysis unless invoked as
`STATIC_ANALYSIS=1 make pre-check`. The full run includes other checks
and runs `govulncheck` across `github.com/juju/juju/...` when that tool
is installed; it can take substantial time and memory. Do not repeat
it for each small, local follow-up when affected tests, scoped lint,
txncheck and PR CI provide the needed feedback. Run it locally when
dependencies or relevant vulnerable call paths change, when an
advisory appears, before release/integration milestones, and
periodically to catch drift that scoped checks and CI might miss.
Review the GitHub checks; a skipped or failing check is not a pass.

## Open reviews: finish on existing branches

Do not recreate these implementations. Verify outstanding review requests,
CI and mergeability; validate any follow-up against the current head.

| Branch / PR | Finding | Remaining action / acceptance contract |
|---|---|---|
| `4.1-model-config-precedence`, #23683 | U7 | Complete review/CI. `v_model_config` and `domain/modelconfig/state` must exclude competing agent stream/version rows; test config values sorting before and after authoritative values and one effective value per key. |
| `4.1-crossmodel-secret-revisions`, #23688 | U11 | Complete remote-query review/CI in `domain/crossmodelrelation/state/model/secrets.go`. Report any-unit-behind secrets, latest revision once, via indexed offer connections. Cover no/current/behind/mixed consumers, multiple secrets/revisions, prefix collisions and filtering; assert revisions and no duplicates. Keep schema/import/recovery/cleanup together. |
| `4.1-operation-query-portability`, #23693 | X34/X33 | Complete review/CI in `domain/operation/state`; keep boolean comparisons and text-size expressions in separate commits. Verify filtering/pruning thresholds across value types. |
| `4.1-nullable-primary-keys`, #23710 | U15, charm | Complete review/CI and land charm first. Required container key, manifest-position key and duplicate-position import diagnostics/tests are implemented; do not redo them. |
| `4.1-application-exposure-scope`, #23723 | U15, exposure | Review against charm, then rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors. Table-specific errors are implemented; assess new-head CI. |
| `4.1-local-secret-revisions`, #23720 | U11 follow-up | Add review regressions in `domain/secret/state/state_test.go`: multiple behind secrets for one unit, empty/nonexistent units, initial-query -> changes-query round trip and explicit nil results. Exercise both `InitialWatchStatementForConsumedSecretsChange` and `GetConsumedSecretURIsWithChanges`; tests must pass with old and portable SQL. |
| `4.1-lookup-key-not-null`, #23724 | U17 | Finish review/CI. A schema-derived decoder-inventory guard and direct `NotValid` classification have been added; retain explicit typed lookup deltas. Independent of both U15 branches. |

### Nullable primary-key review split

The only approved stack is charm #23710 -> exposure #23723. Exposure owns
the combined primary-key guard; lookup #23724 owns an independent INT-key
guard. When exposure and lookup meet, regenerate the combined schema,
remove the global guard's INT exemption (retain only single-column INTEGER
rowid keys), and run both guards. Do not tighten it before lookup keys land
or add the other concern to an independent PR. Both U15 fixes are required
for PoC bootstrap reintegration; charm alone is insufficient.

Remote-secret migration contract: preserve valid tracking; warn/discard
only unresolvable revision-tracking metadata, including malformed names.
Never generalize this exception to secret content, grants or core model data.

## New implementation tasks

Names below are proposed branches, not permission to modify existing owners'
worktrees. Start independently from refreshed `B`, except the secret task's
review sequencing noted below. Confirm each site is still outstanding.

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

### `4.1-dbaccessor-pool-cleanup` — U14

`internal/worker/dbaccessor/tracker.go` leaks pools opened but not retained.
Close them on foreign-keys pragma failure, model-initialization failure and
replacement-pool reopen ping failure; preserve ownership of retained pools.
Inject each failure in `tracker_test.go`, assert every unretained pool closes
and successfully transferred pools remain usable. Upstream is not started;
the register's PoC implementation is not an upstream merge.

### `4.1-agentbinary-conflict` — U16 / X18

Both `domain/agentbinary/state/{controller,model}/state.go`
`RegisterAgentBinary` paths SELECT after a primary-key error, which aborts
a PostgreSQL transaction. Use `ON CONFLICT DO NOTHING`, then compare,
preserving `AlreadyExists` / `AgentBinaryImmutable`, retries and idempotency.
Test new, repeated and different-object registrations in both state suites;
verify the conflict path leaves the transaction usable.

### `4.1-lease-pin-conflict` — X18

`domain/lease/state` `PinLease` ignores a unique violation. Resolve competing
pins in SQL with `ON CONFLICT DO NOTHING`, preserving the existing
already-pinned result. Test competing pins and continued transaction use.
This is portability, not a current dqlite defect; keep production timing
changes out of this branch.

### `4.1-secret-latest-revision`

After the consumed-secret reviews (#23688/#23720), independently rewrite
five queries in `domain/secret/state/state.go`: `updateSecret`,
`GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and
`getSecretsRevisionExpiryChanges`. Explicitly select the latest revision
instead of relying on ungrouped columns beside MAX. Existing and new SQL
must pass regressions with multiple revisions, expiry/rotation rows and
both owner kinds. This does not duplicate #23720's two local queries.

## Lookup-key NOT NULL task (separate session)

Continue existing independent #23724, not a new implementation branch.
Scope: single-column `id INT PRIMARY KEY NOT NULL` (48 model, 17 controller
at `8369e6e2a9`), generated scalar IDs and consumed payload boundaries.
Recheck inventory at the actual base; leave INTEGER rowid keys, composite
keys, historical DDL, 4.0.12 export types and unrelated nullable fields alone.
Do not replace explicit schema constraints with generator inference.

Remaining review/acceptance checklist:

- Keep the explicit typed-delta methods: a generic loop still needs an ID
  accessor and field-mapping closure in every method, obscuring each
  conversion. `requiredLookupID` already centralises NULL rejection.
  Exercise every consumed legacy lookup type's full row, multiple rows,
  nullable non-ID fields, nil IDs and valid zero. Reject missing IDs with
  table/row context and invalid-payload classification; never invent zero.
  No legacy controller delta is needed.
- In `domain/export/payload.go`, validate consumed 4.1.0 model/controller
  raw YAML IDs before scalar decoding loses NULL/omitted versus zero.
  Test valid zero/non-zero, NULL and missing IDs at the actual boundary;
  do not broaden validation to unrelated scalar fields.
- Preserve import exclusions for `change_log_edit_type` and
  `change_log_namespace`, valid seed IDs and seeded-import conflict handling.
  Test representative legacy decode -> transform -> import -> export
  persistence, including existing seeds and a valid non-seed where allowed.
- Verify fresh model/controller schemas reject NULL keys and the complete
  affected declaration/generated-field set is correct. Exercise generated
  recovery decode/replay, valid dump round trips, NULL/missing rejection
  and failure atomicity without relaxing immutable-table requirements.
- Run affected schema/export/import/recovery/cloud-state race suites,
  focused stress/lint/build and regeneration-idempotence checks. Keep the
  INT-key guard independent; reconcile the global guard as described above.
  Stop for release-policy changes, new nullable-key writers or incompatible
  representations. No PostgreSQL-only work belongs in this review.

## Candidates needing bounded scope before dispatch

These register findings are not ready-made assignments. Identify remaining
production paths, behaviour contracts and focused tests before branching;
split conflicts by operation and preserve retries, idempotency and errors.

| Finding | Candidate scope / next action |
|---|---|
| X7 | `GetModelConnectionInfo`: replace IIF with CASE. Identify its production path and fixtures; verify identical NULL/value behaviour. Not proposed upstream. |
| X24 | Replace SQLite `x IN cte` with explicit SELECT membership subqueries. Inventory remaining sites and bound tests/file ownership. Not proposed upstream. |
| X3/X4 | Explicit upsert targets (`domain/sequence`) and qualified conflict expressions. Identify remaining operations; keep distinct from landed objectstore and queued agentbinary/lease work. |
| U11 watcher follow-up | Per-offer-connection remote watcher scoping is separate from #23688's offered-application scope. Agree the event/isolation contract and tests before dispatch. |
| X33 stored-bytes follow-up | A stored-bytes metric such as `pg_column_size` needs backend-aware state code; #23693's text-size work does not implement it. Scope separately, not as a mechanical rewrite. |
| X16 dbrepl output follow-up | Feedback on juju/juju#23730 suggests extracting the duplicated row scanning and tabwriter output from `changeLog` and `changeStream` in `internal/worker/dbrepl/worker.go` into a shared helper taking the query and arguments. Preserve each command's SQL, output and error handling; check current upstream code and exercise both commands. Keep separate from the latest-row SQL task. |

Not stock-upstream tasks: the PostgreSQL `constraint.allocate_public_ip`
export-type mismatch needs backend-aware generation; the intentional
`TestSetAPIAddressesSharedPeerRollsBack` NOT NULL failure is not a dqlite
bug. Keep both recorded rather than reopening them as upstream fixes.

Separate documentation task: `4.1-pr-description-guidance` has a staged
[patch](4.1-pr-description-guidance.patch) and
[PR description](PR-4.1-pr-description-guidance.md), but no branch/PR.
Review the description/comment distinction independently of SQL changes.

## PoC reconciliation handoff

Do not implement PoC integration in stock-upstream worktrees. After landing,
hand merged revisions to the integration owner using UPSTREAM-INTEGRATION.md
and update the register. Explicit outstanding obligations: both U15 fixes,
U17 guard integration, independent reconciliation of #23688/#23720 despite
existing PoC cherry-picks, and #23694's UUID-conflict sentinel (not content
already stored). Keep all other landed-fix reconciliation status in the
register; upstream landing alone does not mark it complete.

## Dispatch

Supply the upstream base, one named branch/worktree, allowed paths,
expected behaviour, prerequisites and required regression cases. The
executor reads this guide and the target repository's instructions; task
selection does not authorise work on other branches.

Example assignment:

> In stock upstream Juju 4.1 at <base>, implement the dbaccessor pool-cleanup
> task from UPSTREAM-PLAN.md on <branch> in <worktree>. Use only that
> checkout's code and test infrastructure. Add the specified regressions,
> run its normal validation, and report the diff and results. Stop for
> unapproved semantics or dependencies.

## Upstream validation and review

- Reproduce the upstream defect where applicable, then add the regression
  alongside the fix. For a behaviour-preserving SQL improvement, assert the
  existing contract across relevant edge cases.
- Run focused tests first, then affected package suites for the assigned
  paths. Preserve unrelated behaviour; do not expand into landed fixes.
- Use the target checkout's standard test commands, dependencies and
  database fixtures. Include actual dqlite checks where required, and its
  prescribed race/stress, watcher-readiness, formatting and lint checks.
- For schema changes, confirm the release's base-DDL/incremental-upgrade
  policy. Use `make rebuild-export` from each branch's own schema; regenerate
  after conflicts from the combined schema, not whole-file selection.
  Exercise writers -> storage -> export -> legacy import -> consumers,
  recovery and supported upgrades. Generation alone is not acceptance;
  generated output must be limited to intended changes.
- Record the base, final commit, exact commands/build tags and results.
  State tests not run and unresolved risks; do not claim broader coverage.
  Explain the upstream defect/benefit and any behaviour change in the PR,
  following the target repository's template.

Return the branch, commits, test evidence and review link to the coordinator.
The coordinator updates shared planning records; parallel agents need not
edit them. Respect the host's test resource budget and other owners' state.
Each independent review targets upstream 4.1; rebase/regenerate when needed
after related changes land, without combining unrelated reviews.
