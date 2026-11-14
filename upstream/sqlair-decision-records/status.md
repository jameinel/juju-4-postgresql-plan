# sqlair-decision-records

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Planned 2026-11-14. Unclaimed. Target repository:
[canonical/sqlair](https://github.com/canonical/sqlair), base `main` at
`bd54c49` when planned. Not a separate PR: these commits are shared, and
both [`sqlair-dollar-placeholders`](../sqlair-dollar-placeholders/status.md)
and [`sqlair-placeholder-style`](../sqlair-placeholder-style/status.md)
carry them, ahead of their own changes. Each branch then adds its own
placeholder decision record.

Next actions:

- [ ] Whichever placeholder session starts first writes these commits
  and records their hashes here and in both branch records; the other
  cherry-picks them.
- [ ] Confirm the backfill list and each record's wording with the user
  before committing; backfilled records describe other people's
  decisions and must cite their PRs.
- [ ] The PR description proposes the convention to the maintainers and
  says it can be dropped from the PR if they decline.

## Decisions (user, 2026-11-14)

- Location: `docs/decisions/`, excluded from the Sphinx build, so it is
  read on GitHub and not published on the docs site. sqlair's docs follow
  Diátaxis, and a decision log is not one of its four types.
- Delivered as shared commits in both placeholder branches, not as its
  own PR.
- Backfill earlier sqlair decisions.

## Design

- Format: lightweight ADRs (Michael Nygard's format): title, status,
  date, context, decision, consequences; plus a "References" list of PRs
  and issues. Statuses: Proposed, Accepted, Superseded by NNNN.
- Files: `docs/decisions/NNNN-short-title.md`, numbered in the order
  they are written. Backfilled records come first, in the order of the
  original decisions, and give the original decision date and PR.
- `docs/decisions/README.md`: purpose, when to write a record, the
  template, and an index table (number, title, status).
- Build exclusion: add `'decisions'` (or `'decisions/*'`; check which
  form Sphinx needs) to `custom_excludes` in `docs/custom_conf.py`
  (lines 186-188). Without it, `-W` in `docs/Makefile.sp:72` makes the
  orphan pages fail the docs build.

## Backfill candidates

Read each PR's discussion before writing; drop any whose rationale can't
be found. Candidates, from `git log` on `bd54c49`:

| Decision | References |
|---|---|
| Named `@sqlair_N` input parameters | #43 (no commit body; check the PR) |
| Remove the prepared-statement cache (finalizer overhead, one-connection limit, unproven benefit) | #128 |
| Reintroduce a driver prepared-statement cache, keyed by statement and DB; transactions only reuse cached statements | #149, #117, #177 |
| `GetAll` overwrites rather than appends to the caller's slice (behavior change, needs a version bump) | #192 |
| Don't track transaction done status; rely on `database/sql` | #191 |
| Nil and empty slice inputs (`IN ()` today; see `sqlair-empty-in`) | #130, #136 |
| Bulk insert and `omitempty` in `INSERT` | #142, #145 |

The first record should be the decision-log convention itself ("Record
design decisions"), so the placeholder record follows the backfill.

## Commits

1. `docs: add a decision record log`: `docs/decisions/README.md` with the
   index and template, `0001-record-design-decisions.md`, and the
   `custom_excludes` entry.
2. `docs: backfill earlier design decisions`: one record per confirmed
   candidate above, and the index updated.

## Validation

- `make -C docs html` (or the starter-pack target used in CI) passes
  with `-W`; no `decisions/` pages are built.
- Check that the CI spelling, link, inclusive-language (woke) and
  Markdown lint jobs either skip `docs/decisions/` or pass on it; add
  words to `docs/.custom_wordlist.txt` if spelling still scans it.
- Links in the records resolve on GitHub.
