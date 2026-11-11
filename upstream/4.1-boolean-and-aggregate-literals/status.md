# 4.1-boolean-and-aggregate-literals

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership/worktree: not assigned by this migration; confirm before acting.

PoC implementation on Stage 5 (`30e2bca25a`, `ada2a852ee`) and Stage 6 test SQL (`fa75871904`); scoped in UPSTREAM-PLAN.md

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

### Upstream register snapshot

`4.1-boolean-and-aggregate-literals` | U20 | not proposed |  | PoC implementation on Stage 5 (`30e2bca25a`, `ada2a852ee`) and Stage 6 test SQL (`fa75871904`); scoped in UPSTREAM-PLAN.md

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

#### `4.1-boolean-and-aggregate-literals` — U20, portability

`domain/network/state` `GetMachineAppBindings` and `GetUnitAPIAddresses`
compare booleans with `0`/`1`; use `FALSE`/`TRUE`. `domain/agentpassword`
`MatchesMachinePasswordHashWithNonce` selects `mci.instance_id` beside
`COUNT` without grouping; use `MAX(mci.instance_id)` (a machine has at
most one cloud instance row). PoC `30e2bca25a`, `ada2a852ee`; the
agentpassword test SQL fix is Stage 6 `fa75871904`.

## U20 provenance snapshot (2026-11-11 migration)

Fixed, Stage 5, `30e2bca25a` (`FALSE` / `TRUE`) and `ada2a852ee` (`MAX(mci.instance_id)`); test SQL `fa75871904` (Stage 6; was `283a00f630` before the 2026-11-06 restack). Not proposed.
