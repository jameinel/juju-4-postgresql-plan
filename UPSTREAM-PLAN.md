# Stock upstream Juju 4.1 remaining work

Use this standalone queue for a stock upstream 4.1 checkout. Read the shared
rules, assigned task and target repository's AGENTS.md/contribution guidance;
no PoC checkout, configuration or test setup is required.

Check upstream code, GitHub reviews and CI before acting; implementation,
landing and PoC reintegration are different states. Keep this document
focused on pending actions.
[UPSTREAM-DONE.md](UPSTREAM-DONE.md) records implemented work and rationale;
[UPSTREAM-FIXES.md](UPSTREAM-FIXES.md) owns finding/provenance/status mappings.
Neither is prerequisite reading for an assigned task.

Status checked against GitHub on 2026-11-04: 9 open reviews and 14 merged
PRs. Current heads, merge revisions and the CI snapshot are in
[the register](UPSTREAM-FIXES.md#upstream-review-status); implemented scope
and historical validation are in UPSTREAM-DONE.md.

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
| `4.1-model-config-precedence`, [#23683](https://github.com/juju/juju/pull/23683) | U7 | Review follow-ups are published and CI is green; obtain remaining approval. Preserve authoritative agent-table values, import filtering and one effective value per key. |
| `4.1-crossmodel-secret-revisions`, [#23688](https://github.com/juju/juju/pull/23688) | U11 | Connection-scoped uniqueness and cleanup follow-ups are implemented; the refreshed head is mergeable and CI is running. Assess that head and obtain approval. Keep any-unit-behind semantics, indexed connections, schema/import/recovery/cleanup and regressions together. |
| `4.1-operation-query-portability`, [#23693](https://github.com/juju/juju/pull/23693) | X34/X33 | Commit identity/message and rebase requests are addressed; submission checks pass. Triage default-bases smoke and cancelled macOS CI, then complete review. Preserve filtering and the reviewed NULL-as-zero column-size fix, which changes the old whole-row-ignored estimate. |
| `4.1-nullable-primary-keys`, [#23710](https://github.com/juju/juju/pull/23710) | U15, charm | Finish review and resolve the cancelled macOS/aggregate CI failure; land charm first. Required container key, manifest-position key and duplicate-position diagnostics/tests are implemented. |
| `4.1-application-exposure-scope`, [#23723](https://github.com/juju/juju/pull/23723) | U15, exposure | Finish review and triage bundles smoke. Rebase/regenerate onto `4.1` after charm lands. Preserve wildcard API/view semantics, legacy scope split/deduplication, reset-to-alpha and deletion rollback/errors. |
| `4.1-lookup-key-not-null`, [#23724](https://github.com/juju/juju/pull/23724) | U17 | CI is green; obtain approval. Schema-derived decoder inventory, direct `NotValid` classification and the transaction-test fix are implemented. Retain explicit typed deltas and the 65-key scope; later reconcile the combined guard with exposure. |
| `4.1-agentbinary-conflict`, [#23728](https://github.com/juju/juju/pull/23728) | U16/X18 | Implementation and regressions exist in both state packages. Triage default-bases smoke and obtain approval. Preserve `AlreadyExists` / `AgentBinaryImmutable`, idempotency and transaction usability. Do not recreate the fix. |
| `4.1-lease-pin-conflict`, [#23729](https://github.com/juju/juju/pull/23729) | X18 | Implementation, pin-contract documentation and retry-safe test-helper fix are published. Triage bundles/revision smoke and obtain approval. Preserve competing-pin, repeated-pin and missing-lease behaviour. |
| `4.1-sequence-upsert-target`, [#23731](https://github.com/juju/juju/pull/23731) | X3/X4 | CI is green with one approval; obtain remaining approval. Explicit namespace conflict targets and qualified updates are implemented. Preserve zero-based allocation, contiguous batches, namespace isolation, rollback and import continuity. |

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

### `4.1-secret-latest-revision`

After the remaining consumed-secret review (#23688), independently rewrite
five queries in `domain/secret/state/state.go`: `updateSecret`,
`GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and
`getSecretsRevisionExpiryChanges`. Explicitly select the latest revision
instead of relying on ungrouped columns beside MAX. Existing and new SQL
must pass regressions with multiple revisions, expiry/rotation rows and
both owner kinds. This does not duplicate the two local queries landed in
#23720.

## Lookup-key NOT NULL task (separate session)

Continue existing independent #23724, not a new implementation branch.
Scope: single-column `id INT PRIMARY KEY NOT NULL` (48 model, 17 controller
at `8369e6e2a9`), generated scalar IDs and consumed payload boundaries.
Recheck inventory at the actual base; leave INTEGER rowid keys, composite
keys, historical DDL, 4.0.12 export types and unrelated nullable fields alone.
Do not replace explicit schema constraints with generator inference.

Implementation, regression coverage and recorded live migration QA are in
UPSTREAM-DONE.md. Remaining work is review/CI and later combined-guard
reconciliation. Preserve this acceptance contract on further changes:

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
| X3/X4, other upserts | Sequence allocation is implemented in #23731. Inventory any other remaining operations before dispatch; do not recreate that fix or combine unrelated conflict paths. |
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
U16's agent-binary fix, U17 guard integration, reconciliation of landed
#23720 and #23726 against
their existing PoC implementations, independent reconciliation of #23688
after it lands, and #23694's UUID-conflict sentinel (not content already
stored). Keep all other landed-fix reconciliation status in the register;
upstream landing alone does not mark it complete.

## Dispatch

Supply the upstream base, one named branch/worktree, allowed paths,
expected behaviour, prerequisites and required regression cases. The
executor reads this guide and the target repository's instructions; task
selection does not authorise work on other branches.

Example assignment:

> In stock upstream Juju 4.1 at <base>, implement the developer-tooling
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
