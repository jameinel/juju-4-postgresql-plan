# Pilot: does the AGENTS.md commit/PR guidance produce more useful output?

Date: 2026-11-17. Session `9dc72382-438f-4e0d-9b0d-766a96c21bf4`. The
harness (`build.sh`, `build-t4.sh`, `run.sh`, `score.py`, `judge.py`), fact
sheets, author notes, raw outputs and judge transcripts (about 1.8 MB) are
not copied here. They remain in that session's local `files/pilot/`
directory.

## Design

- Three merged juju/juju fixes were replayed: #22296 (network-get with a
  relation), #22095 (relation removal and FK errors) and #23101
  (deterministic agent tarball hash).
- Each run used a shallow clone at the PR base with no remote, no other refs
  and no PR head objects. The real PR diff was applied uncommitted.
- The prompt asked the agent to commit the change and write `pr.md`. It
  included a short author note with the non-diff facts and the linked issue
  text. It said not to run builds or tests.
- Arm A used the guidance on upstream/4.1 (`AGENTS.md` plus
  `agent-commit.instructions.md`, the template and `CONTRIBUTING.md`). Arm B
  used this branch.
- All other agent instruction files were identical across arms. Personal
  instructions were isolated with a clean `COPILOT_HOME`, and
  `copilot instruction list` confirmed only repository instructions loaded.
- Writers were claude-sonnet-5.5 and gpt-6.1-terra via Copilot CLI 1.0.112
  in `-p` mode: 12 runs, 15–60 s each, 76 AI credits in total.
- Push, fetch, `gh`, curl, wget and web fetch were denied.
- The judge was gemini-4-argon, blind to arm, scoring each pair in both
  orders (12 judgments). It saw the writer's inputs, the diff and a fact
  sheet written before any run, but neither arm's guidance.

## Results

| Pair (task, writer) | Preferred (both orders) |
| ------------------- | ----------------------- |
| T1 Claude           | A                       |
| T1 GPT              | B                       |
| T2 Claude           | B                       |
| T2 GPT              | B                       |
| T3 Claude           | B                       |
| T3 GPT              | B                       |

Judge totals over 12 judgments per arm (higher is better):

| Measure                       | Max | A  | B  |
| ----------------------------- | --- | -- | -- |
| Fact recall                   | 72  | 69 | 72 |
| QA reproducible               | 24  | 14 | 24 |
| QA setup                      | 24  | 16 | 24 |
| QA total (5 criteria)         | 120 | 96 | 120|
| PR usefulness (1–5)           | 60  | 50 | 58 |
| Commit usefulness (1–5)       | 60  | 56 | 59 |

Verified by reading the outputs:

1. QA for GPT: Arm A wrote prose (3/3), for example "The author verified
   this on LXD…" or numbered steps without commands. Arm B wrote runnable
   `juju` commands with an expected result (3/3).
2. Discriminating QA for Claude: Arm B stated the outcome both without and
   with the change (3/3); Arm A did so 0/3.
3. Unit tests in QA: Arm A listed `go test` or unit tests in QA 4/6; Arm B
   0/6, matching "unit test commands only when no functional path exists".
4. Defect: T3 Claude Arm A's QA bootstraps `dst` before `add-model`, so the
   model is created on the wrong controller and the migrate step fails.
5. Regression: T1 Claude Arm B kept a template HTML comment and a `~JUJU-~`
   placeholder despite the new SHOULD to remove them. Arm A had no such
   leaks (0/6). This is the judge's one preference for A. The cleanup rule
   is not shown to help at this sample size.
6. Gap in both arms: GPT never wrapped commit bodies (6/6; longest lines
   216–311 columns), while Claude kept them at 73 or under. Post-guidance
   human commits keep prose at 72 or under in 79% of cases; commitlint only
   warns above 100. Addressed below.
7. Behaviour note (not scored): T1 Claude Arm A committed only the fix hunks
   and left drive-by refactors uncommitted. Arm B committed everything and
   called out the incidental tidy-ups.

The commit messages are similar across arms; most of the difference is in
PR descriptions and especially QA.

## Rerun with the wrap rule

I added "Wrap body lines at 72 columns." to the Arm B body guidance (branch
commit `ec62079c8c`). I then reran the three GPT Arm B runs (15 AI credits)
and re-judged them against the same Arm A outputs. The first-run outputs and
judgments are in `out/v1/`, `judge/v1/` and `judge/results-v1.json`.

- Longest commit body lines fell from 232–311 to 70–71 columns.
- The judge still preferred B in 3/3 pairs, in both orders. The B QA scores
  stayed at the maximum (30/30).
- The rerun T3 output omitted the "series of migration CI fixes" context
  (F5), which the first sample included. Fact recall varies between samples.
- The rerun T1 QA is runnable but also restates what the author checked.

## Docs-only follow-up (T4)

After a separate review, the guidance gained a commit type list, a
before/after contrast limited to behaviour changes, fenced QA commands, and
"When no functional path exists, such as for documentation-only changes, say
so and why". T4 replays the docs-only #22842 (3.6 to 4.0 upgrade guide:
`juju show-unit` `application-data` orientation) with the draft guidance (D,
`f6dc5ea29d`) and the current guidance (C, `0d9ea6a7c7`). Only `AGENTS.md`
differs. Both arms keep the base's `docs/agents/` rules, which hold no
commit or PR guidance. Six runs (four, then two for arm E below), scored by
script and by reading; no judge.

| Run      | QA                                                  | Leaks             | Body width |
| -------- | --------------------------------------------------- | ----------------- | ---------- |
| D Claude | Read page; fenced `juju` commands verifying the claim, with expected result | `#19267`, `JUJU-` (struck) | 70 |
| D GPT    | Read page and check numbering (prose)               | none              | 220        |
| C Claude | "No functional path"; read page and check numbering | none              | 67         |
| C GPT    | "No functional path"; read page                     | none              | 71         |

Arm E (`40084b3bee`) rewords the clause to: "For documentation changes,
build and view the changed pages and, where practical, check the behaviour
they describe. When no functional path exists, say so and why".

| Run      | QA                                                  | Leaks             | Body width |
| -------- | --------------------------------------------------- | ----------------- | ---------- |
| E Claude | "No functional path"; read page and check numbering | `#19267`, `JUJU-` (struck) | 71 |
| E GPT    | Fenced `make serve` with URL and what to check; fenced `juju` commands on 3.6 and 4.0 with expected results | none | 72 |

- The new clause worked as written: both C runs said there was no functional
  path and invented no setup. Neither arm listed unit tests.
- It also removed the most useful step. D Claude's `juju` commands were not
  manufactured: they check the behaviour the page documents. C Claude
  dropped them.
- No D or C run gave a docs build command (`make -C docs html` or
  `make -C docs run`) to check the rendered page.
- With E, GPT gave the best QA of the six: a correct `make serve` step
  (`dirhtml` on port 8000) plus a behaviour check with expected results.
  Claude still declared no functional path, as C Claude did, so the
  rewording helped one model in one sample. E is no worse than C on any
  check.
- D GPT did not wrap despite having the wrap rule; GPT has now wrapped in 5
  of 6 runs with the rule.
- Fact recall was complete in all four. Both D runs used a "Before this"
  form; neither C run did, but both C bodies still stated the problem and
  impact, so nothing was lost.
- One sample per cell; treat this as a signal about the example's wording,
  not a measurement.

Arm F (`226d85ae23`) limits strikethrough to checklist items, and asks for
the Links placeholders to be replaced with real references or `None`; the
template's Links comments say the same. Across the pilot, 10 of 18 runs had
struck through Links text, and Claude runs had left struck `#19267` and
`JUJU-` placeholders in D and E.

| Run      | QA                                                  | Leaks             | Body width |
| -------- | --------------------------------------------------- | ----------------- | ---------- |
| F Claude | "No functional path"; read page and check numbering | none (`None`)     | 70         |
| F GPT    | Fenced `make -C docs html`, correct `dirhtml` output path, what to check; fenced `juju` commands on 3.6 and 4.0 with expected results | none (`None`) | 69 |

- Both runs wrote `**Issue:** None` and `**Jira card:** None`, with no
  strikethrough or placeholder text in Links.
- Checklist strikethrough still worked. GPT kept the `[ ]` boxes; Claude
  dropped them and ticked "Code style" for a docs-only change.
- QA matched arm E: GPT built the docs and checked the documented
  behaviour; Claude again declared no functional path.

## Caveats

- Small n: 6 pairs, one sample per cell, two writer models, one judge
  model, and Copilot CLI only.
- The judge saturates; many Arm B and Claude scores are at maximum.
- I wrote the fact sheets and author notes, which are aligned with the
  guidance's emphasis on rationale and QA.
- The models may have seen these merged PRs. Word 8-gram overlap with the
  real descriptions is at most 0.13, and the T2 overlap comes from QA
  commands copied from the issue in both arms.
- The judge flagged migration QA steps as unsupported. They come from the
  PR template's instruction for facade changes, and affect both arms
  equally.
