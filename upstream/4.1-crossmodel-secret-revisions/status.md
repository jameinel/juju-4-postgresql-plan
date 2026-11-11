# 4.1-crossmodel-secret-revisions

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Await requested human review; bot findings were answered. Keep any-unit-behind semantics, indexed connections, schema/import/recovery/cleanup and regressions together; reconcile the PoC after landing.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-crossmodel-secret-revisions`, #23688 | U11 | Any remote unit behind makes a secret reportable; return its latest revision once through indexed offer connections. Tracking uniqueness and upserts are scoped by `(offer_connection_uuid, secret_id, unit_name)`; synthetic-unit cleanup uses one CTE-backed DELETE.

From `UPSTREAM-FIXES.md`:

`4.1-crossmodel-secret-revisions` | U11 | [#23688](https://github.com/juju/juju/pull/23688) | `aa6cad849c` | open; current macOS and aggregate checks passed; human review required

### Upstream register snapshot

U11 | latent | `domain/crossmodelrelation/state/model/secrets.go` (two queries) | Group by `sruc.secret_id` but select `sr.uuid` and test `sruc.current_revision` in `HAVING`, so with several consuming units SQLite picks an arbitrary unit row (DIALECT.md X16). Also uses `instr` (X30). | Implemented upstream on open [PR #23688](https://github.com/juju/juju/pull/23688), branch `4.1-crossmodel-secret-revisions`, head `aa6cad849c`. A secret is reported if any remote consuming unit is behind, and the initial query returns its latest revision once through indexed offer connections. Tracking uniqueness/upserts use `(offer_connection_uuid, secret_id, unit_name)`; synthetic-unit cleanup uses one CTE-backed DELETE. The independent local-consumer fix, [PR #23720](https://github.com/juju/juju/pull/23720) (`4.1-local-secret-revisions`, head `ee07d7ccb5`), merged 2026-11-04 as `edb2aef5b9`; review regressions and redundant-DISTINCT removal are complete. Earlier versions were cherry-picked onto `pg-poc/05-queries` on 2026-10-26 (`8f2c49c355` / `79f2e66049`, matching patch IDs then). #23720 was reconciled 2026-11-05 (PoC `f2b508d9fb` dropped; see the reconciliation record). The #23688 cherry-pick is kept as `2f81195ea3`; reconcile it when #23688 lands. Historical PostgreSQL evidence at `08=c5f185e6f6`: `TestModelSecretsSuite` 29/29 and two new watcher tests pass; two of five local consumed-secret tests pass, while three fail in fixtures on separate secret-state X16 sites.

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Secret-query decisions

The #23688 remote fix keeps the any-unit-behind query, indexed
`offer_connection_uuid` schema/export/import/recovery changes, consumer
cleanup and regressions together. Covering only a single consuming unit
would not establish the contract.

Migration warns and discards unresolvable remote revision-tracking rows,
including malformed names. These memoise consumed revisions; they are not
secret content, grants or core model structure. A normal read recreates
missing tracking at the latest revision; dropping tracking does not itself
emit a watcher event. Preserve valid tracking. This is a bounded exception
for reconstructible metadata, not permission to discard inconsistent model
data generally.

The extracted local portability fix owns only
`InitialWatchStatementForConsumedSecretsChange`,
`GetConsumedSecretURIsWithChanges` and
`TestConsumedSecretsChangesMultipleUnitsAndRevisions` in
`domain/secret/state`. It was split onto `4.1-local-secret-revisions` on
`9ea82c03bd`, commit `369fb64c2d`, and published as #23720. The regression
passes with original SQLite and portable SQL; full secret-state race tests
and focused lint passed. The remote review follow-up was `29fc497b6e`;
current recorded PR heads are in the respective branch records. Remote schema-dependent
fixtures stay in #23688. Its watcher still scopes by offered application;
per-connection watcher scoping is separate follow-up.

Earlier versions of both patches were cherry-picked into `pg-poc/05-queries`
on 2026-10-26 with matching patch IDs. That does not establish reconciliation
with #23720's landed head or the eventual #23688 merge. The register records
PostgreSQL results and the separate remaining secret-state fixture failures.

## U11 provenance snapshot (2026-11-11 migration)

Implemented upstream on open [PR #23688](https://github.com/juju/juju/pull/23688), branch `4.1-crossmodel-secret-revisions`, head `aa6cad849c`. A secret is reported if any remote consuming unit is behind, and the initial query returns its latest revision once through indexed offer connections. Tracking uniqueness/upserts use `(offer_connection_uuid, secret_id, unit_name)`; synthetic-unit cleanup uses one CTE-backed DELETE. The independent local-consumer fix, [PR #23720](https://github.com/juju/juju/pull/23720) (`4.1-local-secret-revisions`, head `ee07d7ccb5`), merged 2026-11-04 as `edb2aef5b9`; review regressions and redundant-DISTINCT removal are complete. Earlier versions were cherry-picked onto `pg-poc/05-queries` on 2026-10-26 (`8f2c49c355` / `79f2e66049`, matching patch IDs then). #23720 was reconciled 2026-11-05 (PoC `f2b508d9fb` dropped; see the reconciliation record). The #23688 cherry-pick is kept as `2f81195ea3`; reconcile it when #23688 lands. Historical PostgreSQL evidence at `08=c5f185e6f6`: `TestModelSecretsSuite` 29/29 and two new watcher tests pass; two of five local consumed-secret tests pass, while three fail in fixtures on separate secret-state X16 sites.
