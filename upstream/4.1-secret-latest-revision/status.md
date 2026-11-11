# 4.1-secret-latest-revision

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

Fixed, Stage 5, `5e70fcbc84` (five secret sites, correlated `MAX` subquery, `latestrevision_test.go`), `0ed13ff885` (order revisions), `3a8f0090c2` (unitstate). Upstream: `4.1-secret-latest-revision`.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-FIXES.md`:

`4.1-secret-latest-revision` | U19/X16 | not proposed |  | PoC implementation on Stage 5 (`5e70fcbc84`, `0ed13ff885`, `3a8f0090c2`); extract after the remaining #23688 review

### Upstream register snapshot

U19 | latent | `domain/secret/state/state.go` (`updateSecret`, `GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets`, `getSecretsRevisionExpiryChanges`); `domain/unitstate/state/commithook.go` `updateCharmSecret` | Bare columns beside `MAX(revision)` (DIALECT.md X16). They rely on SQLite's single-`MAX` rule to read the latest revision's row; with other aggregates or several join rows, the bare columns (policy, expiry, owner) may come from any row. The revision listing also had no `ORDER BY` (X17). **Found in Stage 9 run 3 (2026-11-05):** the controller unit's uniter failed with 42803. | Fixed, Stage 5, `5e70fcbc84` (five secret sites, correlated `MAX` subquery, `latestrevision_test.go`), `0ed13ff885` (order revisions), `3a8f0090c2` (unitstate). Upstream: `4.1-secret-latest-revision`.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### `4.1-secret-latest-revision`

After the remaining consumed-secret review (#23688), independently rewrite
five queries in `domain/secret/state/state.go`: `updateSecret`,
`GetRotationExpiryInfo`, `fetchSecrets`, `listCharmSecrets` and
`getSecretsRevisionExpiryChanges`, and the same pattern in
`domain/unitstate/state/commithook.go` `updateCharmSecret`. Explicitly
select the latest revision instead of relying on ungrouped columns beside
MAX, and order revision listings by revision number. Existing and new SQL
must pass regressions with multiple revisions, expiry/rotation rows and
both owner kinds. This does not duplicate the two local queries landed in
#23720.

#23720 has landed; #23688 remains open. Check its current
head and file ownership before dispatching this independent rewrite. A
PoC implementation exists (U19 in UPSTREAM-FIXES.md); an implementer may
read it but must validate on 4.1 independently.

## U19 provenance snapshot (2026-11-11 migration)

Fixed, Stage 5, `5e70fcbc84` (five secret sites, correlated `MAX` subquery, `latestrevision_test.go`), `0ed13ff885` (order revisions), `3a8f0090c2` (unitstate). Upstream: `4.1-secret-latest-revision`.
