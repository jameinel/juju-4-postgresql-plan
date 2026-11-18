# 4.1-pr-description-guidance

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Observed 2026-11-17 22:20 UTC against GitHub.

- Session `9dc72382-438f-4e0d-9b0d-766a96c21bf4` reviewed, amended and
  published this branch on 2026-11-17 at the user's direction. The
  UPSTREAM-PLAN.md claim table does not list an owner; confirm with the
  coordinator before further changes.
- Branch `4.1-pr-description-guidance`, worktree
  `.worktrees/4.1-pr-description-guidance`. Head `226d85ae23` is a single
  signed commit, `docs: move commit and pr guidance into agents.md`, on
  merge base `08fafe7f49`. upstream/4.1 is at `1a0d0c78bc`; the branch has
  not been rebased, and later 4.1 commits touch unrelated files.
- [#23755](https://github.com/juju/juju/pull/23755) is open and
  REVIEW_REQUIRED. anvial's approval is on `b78a42c7c9`, which predates the
  move into `AGENTS.md`. Checks at this observation: 6 passed, 10 skipped,
  and 4 pending (Unit Tests, Checks, JAAS Smoke Test, Documentation build).
- The published body matches [pr.md](pr.md). The 2026-11-17 replies are
  [reply-pilot.md](reply-pilot.md) (comment 6957498004) and
  [reply-docs-qa-and-links.md](reply-docs-qa-and-links.md) (comment
  6962547670). Both were checked byte-for-byte against GitHub.

Next:

1. Await human re-review of `226d85ae23` from anvial and Simon. Evaluate
   any new review before changing the guidance, and describe each change
   as its final state against the previous one. Do not cite review held
   outside GitHub.
2. Confirm the pending checks pass. The change touches only Markdown and a
   redirect in `docs/conf.py`.
3. After landing, delete the local backup branch
   `backup/4.1-pr-description-guidance-pre-squash`. The `AGENTS.md` links
   from the template and `CONTRIBUTING.md` resolve once 4.1 merges forward
   to main.

Not addressed here, candidates for separate changes:

- claude-sonnet-5.5 still answers "no functional path" for a docs-only
  change and asks the reviewer only to read the page, under every wording
  tested.
- `.github/instructions/0-main.instructions.md` §3 still asks for every
  SHOULD deviation to be justified in the PR description and commit
  message. It was left alone: changing it is outside this PR, and none of
  the 24 pilot outputs contained deviation commentary.
- The type bullet lists `perf`, while `fix` still "covers bug and
  performance fixes".

## Review artifacts

Saved text retains its original content and may predate the published PR.
Compare against GitHub before preparing an update; publication needs
explicit approval of the exact content and action.

- [pr.md](pr.md): published body for `226d85ae23`. Its git history holds
  the earlier `15119bb898` draft.
- [reply-pilot.md](reply-pilot.md): the A/B pilot, the 72-column rerun,
  the template-cleanup and QA amendments, and notice of the squash.
- [reply-docs-qa-and-links.md](reply-docs-qa-and-links.md): the type list,
  conditional before/after, docs QA and Links/strikethrough amendments, and
  the docs-only replay.
- [pilot-results.md](pilot-results.md): pilot design, scores, caveats and
  the docs-only follow-up arms D to F.
- [guidance.patch](guidance.patch): historical patch at `15119bb898`,
  superseded by the branch.

## Decisions at `226d85ae23`

The guidance lives in `AGENTS.md` under Commit Messages and Pull Requests,
after General Code Structure Guidelines. At anvial's request, the old
`agent-commit.instructions.md` is deleted. The template, `CONTRIBUTING.md`
and the `docs/conf.py` redirect point to the new sections; the redirect
previously pointed at the deleted file.

- Commit types: the 11 that commitlint's `type-enum` accepts, rather than
  "the standard Conventional Commits types". The spec itself defines only
  `feat` and `fix`.
- Commit body: explain the problem, cause and impact. Contrast before and
  after only when behavior changes, so `docs` and `chore` commits are not
  forced into that form. Wrap at 72 columns; commitlint only warns above
  100.
- PR description: the prospective merge commit message. Review history
  goes in comments. The template remains MUST; the description and QA
  guidance is SHOULD.
- QA: functional and reproducible, with setup, expected results and fenced
  commands, preferably a step whose result differs without the change.
  Documentation QA builds and views the changed pages and, where practical,
  checks the behavior they describe. When no functional path exists, say
  why; only then list unit tests.
- Template text: remove HTML comments. Strikethrough is only for checklist
  items that do not apply. Links take real references or `None`, and the
  template's Links comments say the same. Previously `AGENTS.md` asked for
  every inapplicable item to be struck through, while the template said to
  remove the issue line.

## Evidence

- Corpus: a text heuristic over PRs merged since `AGENTS.md` was added
  (2025-11-21) found that most had a QA section. Only about a third gave
  `juju` commands, about 15% stated an expected result, and a quarter leaked
  template HTML comments into the merge commit. Of 1593 human PRs in that
  period, 39% gave real Links references, 53% had no Links section, 2.5%
  kept a raw placeholder, 1.9% wrote N/A, and none struck placeholders
  through.
- Pilot, three merged fixes × two writers, blind gemini-4-argon judge: the
  draft guidance was preferred in 5 of 6 pairs, mostly for QA. Reproducible
  QA scored 14 vs 24 of 24, and unit tests appeared in QA in 4 of 6 vs 0 of
  6 runs. With the 72-column rule, GPT wrapped in all three reruns.
- Docs-only replay of #22842, one run per model and arm, scored by
  reading:
  - The intermediate "no functional path, such as documentation-only"
    wording dropped the behavior check.
  - Under the final docs wording, GPT gave a correct docs build
    (`make -C docs html` writes `dirhtml`; `make serve` uses port 8000) and
    `juju` checks in both runs. Claude did not.
  - Links: struck through in 10 of 18 earlier runs; `None` in both runs
    after the Links change.
- Caveats: small n, one sample per cell, Copilot CLI only, and the fact
  sheets and notes were written by the same session. See
  [pilot-results.md](pilot-results.md).

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

From `UPSTREAM-DONE.md`:

`4.1-pr-description-guidance`, [#23755](https://github.com/juju/juju/pull/23755) | Agent guidance | Distinguish reviewer context, prospective merge-message content and reproducible reviewer QA from validation history at `15119bb898`.

### Upstream register snapshot

`4.1-pr-description-guidance` | Agent guidance | [#23755](https://github.com/juju/juju/pull/23755) | `b78a42c7c9` | open; await Simon's response about moving guidance to AGENTS.md; 7 checks passed, 12 skipped

<!-- Migrated from UPSTREAM-FIXES.md; historical evidence retains its revision. -->

### Separate agent-workflow guidance

`4.1-pr-description-guidance`: description/comment distinction, unrelated
to SQL fixes. Published as juju/juju#23755 at `15119bb898`.
The [patch](guidance.patch) is historical text. The PR draft then held in
`pr.md` was replaced with the published body on 2026-11-17; see git history
for the earlier draft.
