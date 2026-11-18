Agent guidance for commit messages and PR descriptions moves from
`.github/instructions/agent-commit.instructions.md` into `AGENTS.md`, and the
old file is removed. One file that every agent loads avoids drift between two
copies. The old file was also excluded from Copilot code review
(`exclude-agent: code-review`), so review did not check against it.

In juju the PR description becomes the merge commit body, so both texts end
up in `git log`. A text heuristic over PRs merged since `AGENTS.md` was added
found that most had a QA section. However, only about a third gave `juju`
commands, about 15% stated an expected result, and a quarter leaked template
HTML comments into the merge commit. The guidance therefore asks for rationale
and functional QA, not just format:

- **Commit Messages** keeps the Conventional Commits format that commitlint
  enforces on every PR commit and lists the types it accepts. It adds Juju
  rules commitlint cannot check: type boundaries (`refactor`, `docs`,
  `chore`), single-word scopes, and a body that explains the problem, cause
  and impact, contrasts behavior before and after when it changes, and
  wraps at 72 columns. The generic Conventional Commits explanation is
  dropped.
- **Pull Requests** requires the PR template. It asks for the template's
  HTML comments to be removed, strikethrough only for checklist items that
  do not apply, and real references or `None` in Links. It treats the
  description as the prospective merge commit message and asks for
  reproducible, functional QA steps with setup and expected results,
  preferably a step whose result differs without the change. For
  documentation changes, QA builds and views the changed pages and, where
  practical, checks the behavior they describe. When no functional path
  exists, the description says why. Review history belongs in comments.

The new description and QA guidance uses SHOULD; using the PR template
remains MUST.

A small A/B pilot supports the change. Three merged fixes (#22296, #22095,
#23101) were replayed with Copilot CLI using claude-sonnet-5.5 and
gpt-6.1-terra, once with the 4.1 guidance and once with this change. A blind
gemini-4-argon judge preferred this change's output in 5 of 6 pairs, in both
presentation orders. Most of the difference was in QA. With this change, all
six QA sections gave `juju` commands with an expected result and none listed
unit tests. With the 4.1 guidance, three of six gave commands and four of six
listed unit tests. One run with this change still left a template HTML
comment and an empty Jira placeholder. GPT did not wrap commit bodies with
either guidance. With the 72-column rule added, a rerun wrapped all three,
and the judge still preferred this change in each pair. These runs predate
the type list, the conditional before/after wording and the documentation
QA wording. A follow-up replayed a documentation-only change (#22842). With
the final wording, GPT built the docs and checked the documented behavior;
Claude only asked the reviewer to read the page. Earlier runs often struck
through the Links placeholders; with the final Links wording, both models
wrote `None`.

`PULL_REQUEST_TEMPLATE.md` and `CONTRIBUTING.md` link to the new sections,
and the template's Links comments ask for `None` when there is no issue or
Jira card. The documentation redirect for the old conventional commits page
points to `AGENTS.md#commit-messages`. The links resolve once this is merged
forward to main.

## Checklist

- [ ] ~Code style: imports ordered, good names, simple structure, etc~
- [ ] ~Comments saying why design decisions were made~
- [ ] ~Go unit tests, with comments saying what you're testing~
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

Replay #23101 against this guidance. Start from a shallow clone of its base,
so the agent cannot find the merged answer:

```sh
git init -q -b work qa-23101 && cd qa-23101
git fetch -q --depth=50 https://github.com/juju/juju 0a88bbc39b794ec21a0360c76f8e6fd4f17c10bb
git checkout -q -B work FETCH_HEAD
git fetch -q --depth=1 https://github.com/juju/juju pull/23755/head
git checkout FETCH_HEAD -- AGENTS.md PULL_REQUEST_TEMPLATE.md CONTRIBUTING.md
git rm -q .github/instructions/agent-commit.instructions.md
git -c commit.gpgsign=false commit -qm "chore: use guidance under test"
gh pr diff 23101 -R juju/juju | git apply
```

Start a coding agent in `qa-23101` with personal instructions disabled. For
Copilot CLI, point `COPILOT_HOME` at an empty directory and set
`COPILOT_GITHUB_TOKEN=$(gh auth token)`. Give it this prompt:

```text
This checkout of juju/juju contains an uncommitted change that is complete,
reviewed and tested. Do not modify the code and do not run builds or tests.
Commit the change, and write the pull request description you would submit
to pr.md without committing it. Do not push or open a pull request.

Notes from the change's author: no issue for this; it's one of a series of
fixes for the model-migration CI tests. Migrating a model between two
controllers bootstrapped with --build-agent left the target with two agent
binary records for the same version/arch, and new machines failed with
"agent binary info mismatch". Only affects locally built agents
(--build-agent, or bootstrap falling back to a local build), never
production controllers.
```

Expected results:

- The commit body explains that tar header timestamps made identical content
  hash differently, the migration failure this caused, and that only locally
  built agents are affected.
- The QA steps in `pr.md` bootstrap two controllers with `--build-agent`,
  create and populate the model on the source, migrate it, add a unit and
  state that it provisions without an agent binary info mismatch.
- `pr.md` contains no HTML comments, its Links give `None` rather than
  `#19267` or an empty `JUJU-` placeholder, and its QA does not fall back
  to unit test commands.

To compare with the 4.1 guidance, repeat from a fresh directory without the
second `git fetch`, the `git checkout FETCH_HEAD --`, `git rm` and
`git commit` lines.

## Documentation changes

Agent and contributor guidance only: `AGENTS.md`, `PULL_REQUEST_TEMPLATE.md`,
`CONTRIBUTING.md`, and a redirect in `docs/conf.py`. No user-facing
documentation changes.

## Links

**Issue:** None

**Jira card:** None

