# dqlite-returning-errors

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Migrated from existing planning records on 2026-11-11; no new GitHub
or validation check was performed for this migration. Evidence below
retains its original dates and revisions; it does not validate later heads.

Unclaimed follow-up; this directory is a task identifier, not evidence
that a branch or worktree has been created. Confirm scope and reserve
the task in the dashboard before implementation.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### TODO: dqlite error reporting for `RETURNING`

Follow up independently of the targeted keymanager fix merged in
[juju/juju#23756](https://github.com/juju/juju/pull/23756). Other Juju queries,
including the shared sequence allocator, need portable `RETURNING` support;
do not treat this defect as a general prohibition on `RETURNING`.

The default-bases investigation reproduced a dqlite 1.18.6 error-response
bug: `query__batch` writes column metadata before stepping the statement.
When an `INSERT ... RETURNING` encounters a stale-snapshot write conflict,
the failure path appends an error without clearing the metadata. The client
reads the column count as the error code and Sqlair's `_sqlair_0` alias as
the message, hiding the retryable database error.

- [x] Locate the upstream fix:
  [canonical/dqlite#918](https://github.com/canonical/dqlite/pull/918) merged
  into `main` on 2026-11-04 as `366191c0d3f0`. It discards partial query
  responses on error and tests prepared/direct `RETURNING` constraint
  failures. Juju's bundled dqlite 1.18.6 does not contain it.
- [ ] Identify a release or approved backport containing that fix and
  integrate it into Juju's dqlite dependency.
- [ ] Verify Juju's actual dqlite dependency preserves retryable errors from
  `RETURNING` under contention, as well as successful returned IDs and
  zero-row results. Keep SQLite and PostgreSQL portability intact.
- [ ] Review other `RETURNING` call sites for exposure to this defect.
  Reconsider the keymanager insert/select workaround after the dependency
  fix is available and validated.

Related but distinct:
[canonical/dqlite#920](https://github.com/canonical/dqlite/pull/920) merged
on 2026-11-09 and preserves busy errors otherwise misreported as a nonempty
statement tail. Merge details are in UPSTREAM-DONE.md. Identify a release or
approved backport and validate Juju's actual dependency before treating this
path as fixed in Juju; the response-buffer fix alone does not resolve it.

Reproducer: `domain/keymanager/state/state_test.go`,
`TestEnsureUserPublicKeyConcurrentWrite`, using real dqlite. With the original
`RETURNING` implementation it produces the same `_sqlair_0` error as
[the failing CI job](https://github.com/juju/juju/actions/runs/42120009349/job/127569657575).
For now, keep the current fix limited to keymanager's insert followed by a
unique-key lookup in the same transaction; do not restore `LastInsertId`.

<!-- Migrated from UPSTREAM-DONE.md; historical evidence retains its revision. -->

## Landed dependency follow-up (2026-11-09)

[canonical/dqlite#920](https://github.com/canonical/dqlite/pull/920) merged
on 2026-11-09 at 07:29:32 UTC as
`b434264069050ac258646b89a31651385654be61`, with final head
`64251023b0857cf46599ffd3c0681da7e3bcf557`. It preserves busy errors
otherwise misreported as a nonempty statement tail, separately from #918's
partial-response correction. GitHub merge metadata was verified on
2026-11-09; no local validation or Juju dependency integration was performed.
Release/backport selection and validation against Juju's actual bundled
dependency remain in UPSTREAM-PLAN.md.
