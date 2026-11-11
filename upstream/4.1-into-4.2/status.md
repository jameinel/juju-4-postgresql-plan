# 4.1-into-4.2

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Confirm the current owner and state before acting; this migration does
not assign ownership or authorize changes to an existing worktree.

Recorded state: juju/juju#23775 merged on 2026-11-10 at final head
`496c7d7818`, carrying 4.1 through `bd36cef535`. Later 4.1 commits and
mainline IIF cleanup remain separate follow-ups. Live controller and migration
QA were not recorded. See the integration evidence below.

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md)

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## 4.1 to main integration

[#23775](https://github.com/juju/juju/pull/23775) merged on 2026-11-10
at 08:58:40 UTC as `fa398c45ecbc00145b5906284ebae74789cb6846`, with
final head `496c7d78187be3d0425fb66b386e17638872ce4a`. That head merged
4.1 `bd36cef53587f60bbdb4d910eb6338b9d95fba4a` into main
`419d02804e5df3a1facb80c7eb749be7176bfc84`, including agent-binary
registration, backup recovery preflight and resource lookup fixes.
The local merge had no conflicts and matched `git merge-tree`.

Gvantsats and gfouillet approved that head; Van Cleef commented with no
issues and three optional suggestions concerning duplicated test helpers,
the recovery command's terminal error, and trailing YAML diagnostics.
The final-head localhost smoke and aggregate checks passed after an earlier
localhost failure; skipped checks are not passes.

On `496c7d7818`, affected backups, CLI, modelcmd, agent-binary and resource
state suites passed on SQLite and real dqlite with race detection.
`make go-build`, `STATIC_ANALYSIS=1 make pre-check` and `git diff --check`
passed. The first pre-check attempt with `GOPROXY=off` lacked a cached
license-analysis dependency; the normal-proxy retry passed. Live controller
and migration QA were not recorded. Subsequent 4.1 changes and mainline
SQL `IIF` cleanup are not included; their follow-up is in the plan.
