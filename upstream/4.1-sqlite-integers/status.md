# 4.1-sqlite-integers

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

merged 2026-10-30; merge `8369e6e2a9`

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [reply-review.md](reply-review.md)

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-sqlite-integers`, #23680 (U1/U2/U12) | Aligned: correct grouping at application level and change incorrectly declared lookup/FK IDs to INT. Legacy parsing belongs at import. No D55-driven follow-up indicated.

From `UPSTREAM-DONE.md`:

`4.1-sqlite-integers`, #23680 | U1/U2/U12 | Correct application unit counts and integer lookup/relation-status declarations. SQLite coercion hid mismatched TEXT IDs; legacy string IDs are converted and exports use integer types.

From `UPSTREAM-FIXES.md`:

U1/U2 | PoC `367c35c243`; separate review commits `7c96d020d5` / `6b0f24d11c` on `4.1-sqlite-integers`.

### Upstream register snapshot

`4.1-sqlite-integers` | U1, U2, U12 | [#23680](https://github.com/juju/juju/pull/23680) | `0e9be8ddd7` | merged 2026-10-30; merge `8369e6e2a9`

<!-- Migrated from UPSTREAM-FIXES.md; historical evidence retains its revision. -->

### Integer-schema review follow-up (2026-10-29)

U1/U2/U12, juju/juju#23680: stronger conversion/view tests, `NotValid` errors
and developer schema-evolution documentation; merged on 2026-10-30 at head
`0e9be8ddd7` (merge `8369e6e2a9`). PoC reintegration was not part of that
review; it was completed in the 2026-11-05 reconciliation below.
Race-enabled tests, 52 stress iterations and targeted lint passed; live QA
not run. [PR draft](../../archive/pr-descriptions/PR-4.1-sqlite-integers.md) and
[review decisions/evidence](reply-review.md) hold details.

## U1 provenance snapshot (2026-11-11 migration)

Fixed, Stage 4, `fix(schema): correct lookup types and revision unit counts`.

## U2 provenance snapshot (2026-11-11 migration)

Fixed, Stage 4, same commit as U1.

## U12 provenance snapshot (2026-11-11 migration)

Fixed, Stage 6, `fix(schema): declare relation status type ids as integers` (D30). 4.1 export types become `int64`; the 4.0.12 import delta parses the legacy strings.
