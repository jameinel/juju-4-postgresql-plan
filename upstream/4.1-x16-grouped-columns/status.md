# 4.1-x16-grouped-columns

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

- Owner/session: `48eb9888-f082-47eb-a43f-75ac280c4a3e`.
- Branch: `4.1-x16-grouped-columns`.
- Worktree: `.worktrees/4.1-x16-grouped-columns` under the primary checkout.
- Base: `148e09f015fe59237f382b9df1a468c15ee7fa42`.
- Recorded phase: published.

[#23784](https://github.com/juju/juju/pull/23784), head `c3161fb4ff81a08626ce2a6a1ef99f4645ee5c2a`, base `juju/juju:4.1`; human review required and CI running. User authorized reassignment from session `4d032417-af51-4f77-b950-bf70dd707e46`. Commit `c3f75cf032` preserves all CA certificates in both consumers; `0973ebb4af` adds joined-credential regression coverage. Review follow-up `c3161fb4ff` retains clouds with no CA certificate rows, distinguishes absent and stored empty certificates, and updates the duplication comment. Van Cleef's feedback was [answered](https://github.com/juju/juju/pull/23784#issuecomment-6783412624); shared-row refactoring and PostgreSQL test infrastructure are deferred. Affected SQLite/dqlite race suites, scoped lint and txncheck passed; default pre-check skipped static analysis. The user previously reported successful live Kubernetes backend certificate and secret-count checks and reached cleanup; removal details were not independently verified. New no-CA functional QA and PostgreSQL execution were not run. The published PR description contains the updated QA; `pr.md` predates the follow-up. Follow up on multi-certificate integration CI. Do not rebase implicitly.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-PLAN.md`:

`4.1-x16-grouped-columns` | The original portability-only scope required rework because the row converters still discarded CA certificates. **Rework is published** in [#23784](https://github.com/juju/juju/pull/23784), head `c3161fb4ff`: corrected SQL retains the rows and both consumers collect all certificates, including clouds without certificate rows. Preserve the multi-certificate/attribute/auth-type contract; do not revert to a syntax-only patch.

### Upstream register snapshot

`4.1-x16-grouped-columns` | U22/X16 | not proposed |  | Revised local `c3f75cf032` fixes SQL and preserves all CA certificates in both consumers; the older Stage 5 portability-only scope was insufficient. Publication/QA status is in UPSTREAM-PLAN.md.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

#### `4.1-x16-grouped-columns` — U22, portability

Remove ungrouped projections while preserving secret counts and backend
configuration. CA certificates are not functionally dependent on the old
grouping keys: a cloud can have several. Preserve every certificate in both
consumers, deduplicating repeated join rows; this fixes the previous loss of
all but an unspecified certificate.

The original [#17993](https://github.com/juju/juju/pull/17993) addressed
model-specific Kubernetes credentials. The later
[#22027](https://github.com/juju/juju/pull/22027) documents one displayed
backend per model; credential attributes are merged into that backend.
Neither establishes an intent to discard additional cloud CA certificates.

- `domain/secretbackend/state/state.go`: `ListSecretBackends` and the
  in-use Kubernetes backend query group by every selected column;
  `getK8sSecretBackendForModel` uses `SELECT DISTINCT`. PoC `52a0eea6ee`.
- `domain/removal/state/controller/secret.go`
  `getK8sSecretBackendForModel`: `SELECT DISTINCT` instead of
  `GROUP BY vm.name, vcca.attribute_key`; adds
  `TestGetActiveModelSecretBackendK8sBuiltIn`, which had no coverage
  before. PoC `367f754dd6`.

Both `getK8sSecretBackendForModel` copies and the in-use listing must return
all cloud CA certificates, including with multiple credential attributes and
cloud auth types. Validate counts and certificate deduplication on both
SQLite and real dqlite.

Follow up with CI coverage for the end-to-end multi-certificate case: register
a Kubernetes cloud with two CA certificates, create a model using its built-in
secret backend, and verify that the backend listing returns both certificates
and counts secrets rather than revisions. Exercise secret removal as well,
without treating successful removal alone as proof that both certificates were
returned. Consider extending the existing secrets integration suite and ensure
the test actually runs in CI; this is separate from PostgreSQL validation.

## U22 provenance snapshot (2026-11-11 migration)

Stage 5 secretbackend `52a0eea6ee` and removal `367f754dd6` corrected SQL but retained lossy CA converters; revised upstream commit `c3f75cf032` also preserves all cloud certificates. Storage PoC `640d3bcabf` and [#23766](https://github.com/juju/juju/pull/23766), head `65dbcae6a9`, choose the lowest attachment UUID. **Local merge hold and rework required under D55:** the user confirmed the attachment invariant is uncertain; investigate before choosing schema enforcement or a domain-based selection rule. Determinism is not proof of the correct mount. Ownership, publication and QA remain in UPSTREAM-PLAN.md.
