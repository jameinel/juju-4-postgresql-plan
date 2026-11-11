# 4.1-dqlite-nullable-boolean

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Ownership is not established by this historical review entry. Confirm
the current owner before modifying an existing branch or worktree.

Head `a848d4426e` is based on #23756 at `63a93201b5`, now merged. Rebase onto refreshed `4.1`, revalidate and obtain human review. Preserve NULL/false/true and the upstream-fix sentinel.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

### Upstream register snapshot

`4.1-dqlite-nullable-boolean` | Nullable Boolean, dqlite CI | [#23759](https://github.com/juju/juju/pull/23759) | `a848d4426e` | open; temporary SQL casts and full domain CI published; rebase onto `4.1` after merged #23756, revalidate and obtain human review

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### Nullable BOOLEAN results in real-dqlite domain tests

The broader real-dqlite `-race` run for
[juju/juju#23756](https://github.com/juju/juju/pull/23756) exposed three
existing failures where SQL NULL Boolean values are returned as non-null:

- `domain/logging/state`: `TestSetLokiConfigInsecureSkipVerifyNil` expects
  `InsecureSkipVerify == nil`, but receives a non-nil pointer.
- `domain/machine/state`:
  `TestDetachLostMachineCloudInstanceCleansVolumeAndPreservesIntent` and
  `TestDetachLostMachineCloudInstanceCleansFilesystemAndPreservesIntent`
  expect cleared `persistent` / `read_only` columns to scan into invalid
  `sql.Null[bool]` values, but receive valid values.

The root cause is dqlite's response encoding, fixed upstream by
[canonical/dqlite#883](https://github.com/canonical/dqlite/pull/883),
but not in the bundled 1.18.6. This is separate from the `RETURNING` defect.

Published [juju/juju#23759](https://github.com/juju/juju/pull/23759) at
`a848d4426e` as a temporary SQL workaround, based on #23756 at `63a93201b5`.
It casts the logging and provisioner Loki reads, both storage-status
projections, and machine test projections to INTEGER. It re-enables both
excluded packages in the real-dqlite CI step and adds a sentinel that fails
once NULL is preserved by the bundled library. The status output remains
false for both NULL and false; the provisioner's nil Loki pointer retains
the distinction between the default and explicit false.

The refreshed breakout commits `c9a168b5cd`, `c51a0e05a1` and
`39b1dc3cd8` were integrated separately into the PostgreSQL PoC on
2026-11-11; see the [PoC evidence](../../stages/06-compatibility.md#nullable-boolean-integration-2026-11-11).
That integration does not change the upstream PR's review or merge status.

The independent [juju/juju#23765](https://github.com/juju/juju/pull/23765),
head `2e12c3150a` directly on upstream `4.1` at `148e09f015`, preserves
NULL Boolean and datetime/timestamp pointers in controller exports. It
adds `IS NULL` markers to existing queries, following model export, without
changing the backup format. Its transaction retry resets the captured
payload. The worktrees are `.worktrees/4.1-dqlite-nullable-boolean` and
`.worktrees/4.1-controller-export-null` under the primary checkout; both
were clean at these heads.

Validation:

- The three reported failures reproduced on real dqlite. Focused
  NULL/false/true cases passed on real dqlite and SQLite for the Loki
  pointer and machine `sql.Null[bool]` projections.
- At the earlier #23759 snapshot, `make run-go-tests TEST_PACKAGES='./domain/...'
  TEST_ARGS='-p 2 -race -count=1'` passed for 225 test packages with
  `dqlite,libsqlite3` tags in 1067 s. After a final table-alias-only SQL
  edit, the complete logging package passed with `-race` on both backends.
- The sentinel passed with the uncast query and failed with its intended
  removal message when temporarily changed to return a preserved NULL;
  the uncast query was restored.
- Dqlite REPL QA confirmed the stored NULL reads as false uncast, while
  NULL/false/true cast to NULL/0/1.
- Scoped `golangci-lint`, repository `txncheck`, `gci` and
  `git diff --check` passed. `make pre-check` passed with static analysis
  skipped; scoped lint was run separately. PostgreSQL was not tested.
- At the current heads, affected #23759 packages and #23765's controller
  export suite passed on SQLite and real dqlite; dqlite tests used `-race`.
  Export generator tests, regeneration without drift, scoped lint and
  repository `txncheck` passed. The manual backup QA in #23765 is pending.
- On 2026-11-06, #23759's Unit Tests job passed on its second attempt,
  including logging and machine in the full real-dqlite domain step
  (approximately 5m25s). Its upgrade pre-check failed again on Ubuntu 26.04
  at `snap info juju` with `no snap found for "juju"`; the aggregate check
  failed. A local `snap info juju` succeeded, but the runner failure's
  underlying cause is not confirmed. #23765 CI was still running without
  reported failures; its upgrade smoke was skipped.

- [ ] Rebase #23759 onto refreshed `4.1` now that #23756 has landed,
  revalidate and finish review. The 2026-11-09 snapshot reports successful
  upgrade pre-check and aggregate jobs; continue monitoring the real-dqlite
  domain step's runtime on subsequent heads.
- [ ] Finish #23765 review, CI and functional backup QA independently of
  #23759. Retain the distinct NULL, false, true and timestamp cases.
- [ ] Manual QA (user): run #23765's disposable-controller backup procedure
  for head `3b733c237b`; verify NULL/false/true flags and NULL/set lease
  timestamps, then record the results. Lease rows are exported but not
  restored.
- [ ] When Juju bundles a release containing #883, review the temporary
  casts, sentinel and model/controller export NULL markers; sweep nullable
  Boolean reads. Verify NULL, false and true remain distinct where required,
  along with nullable timestamps, on real dqlite and SQLite. Restore
  `constraint.allocate_public_ip` to `BOOLEAN` (DIALECT.md X25).
