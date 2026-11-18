Following the earlier review, I've revised the guidance and tested whether it
improves the output, not just its format.

Changes since the approval:

- Moved Commit Messages and Pull Requests after General Code Structure
  Guidelines, so the index and build/test guidance stay at the top.
- The commit body is still REQUIRED except for trivial changes, and SHOULD
  explain the problem, cause and impact, then contrast before and after. In
  the history I checked, the most useful human messages (#22296, #22095)
  explain the problem and cause; few use the Before/After form.
- Commit bodies should wrap at 72 columns (see the pilot below).
- The template's HTML comments and unused `#19267`/`JUJU-` placeholders
  SHOULD be removed. They currently reach a quarter of merge commits.
- QA steps SHOULD be functional and reproducible, with setup and expected
  results. They should prefer a step whose result differs without the change
  and list unit tests only when there is no functional path.
- Corrected the description: commitlint already enforces the format rules
  (type, subject case, header length), so the section adds the Juju rules it
  cannot check.
- Removed the duplicated "justify a SHOULD deviation" rule, which
  `0-main.instructions.md` §3 already states.
- Fixed the `docs/conf.py` redirect, which pointed at the deleted
  instructions file.

Pilot. I replayed three merged fixes (#22296, #22095, #23101) from shallow
clones at their bases, with the real diff applied uncommitted and a short
author note. claude-sonnet-5.5 and gpt-6.1-terra each committed the change
and wrote a PR description, once with the 4.1 guidance (A) and once with
this branch (B). Personal instructions were isolated. A blind gemini-4-argon
judge compared each pair in both orders against fact sheets written before
the runs:

| Pair      | Preferred |
| --------- | --------- |
| T1 Claude | A         |
| T1 GPT    | B         |
| T2 Claude | B         |
| T2 GPT    | B         |
| T3 Claude | B         |
| T3 GPT    | B         |

| Judge totals (12 judgments each) | A  | B   |
| -------------------------------- | -- | --- |
| Fact recall (max 72)             | 69 | 72  |
| QA reproducible (max 24)         | 14 | 24  |
| QA total (max 120)               | 96 | 120 |
| PR usefulness (max 60)           | 50 | 58  |
| Commit usefulness (max 60)       | 56 | 59  |

What drove the difference:

- With A, GPT's QA was a prose record of what the author did; with B it was
  runnable `juju` commands with expected results (3/3 each way).
- With B, Claude stated the outcome both without and with the change (3/3,
  versus 0/3 with A).
- A put unit tests in QA in 4/6 runs, B in 0/6.
- T3 Claude with A created the model on the wrong controller before
  migrating, so its QA would fail as written.

Against B: in T1, Claude still left a template HTML comment and `~JUJU-~`,
which was the judge's reason to prefer A. Commit messages were similar
across arms.

Caveats: 6 pairs with one sample each, two writer models, one judge, and
Copilot CLI only. The judge saturates, and the models may have seen these
merged PRs, although overlap with the real descriptions is low.

Gap in both arms: GPT never wrapped commit bodies (lines of 216–311
columns), while 79% of recent human commits wrap prose at 72. commitlint only
warns above 100. I added "Wrap body lines at 72 columns." and reran the three
GPT runs with B. All three wrapped at 71 columns or under, and the judge
still preferred B in each pair, in both orders. The rerun T3 dropped the
"series of migration CI fixes" context the first sample had, so recall
varies between samples.

I'll squash the branch to a single commit, because the first three commits
edit a file the fourth deletes. The harness and raw outputs are available if
anyone wants to rerun it with other agents.

