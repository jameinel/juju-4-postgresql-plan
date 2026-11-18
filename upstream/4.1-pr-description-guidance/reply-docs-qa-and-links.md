On further review, I amended the commit with four changes to the guidance:

- The type bullet lists the 11 types commitlint accepts (`type-enum` in
  `@commitlint/config-conventional`). It previously said "the standard
  Conventional Commits types", which was ambiguous because the spec itself
  defines only `feat` and `fix`.
- The before/after contrast is asked for only when the change alters
  behavior. It previously applied to every commit, which pushed `docs` and
  `chore` commits into a form that does not fit them.
- QA commands go in fenced code blocks. For documentation changes, QA
  builds and views the changed pages and, where practical, checks the
  behavior they describe. When no functional path exists, the description
  says why. Previously, the only guidance for changes without a functional
  path was to list unit tests.
- Strikethrough is for checklist items that do not apply. In Links, the
  `Fixes #19267.` and `[JUJU-](…)` placeholders are replaced with real
  references or `None`, and the template's Links comments say the same.
  Previously, `AGENTS.md` asked for items that do not apply to be struck
  through and for unused placeholders to be removed, while the template
  said to remove the issue line. Agents struck through Links text in 10 of
  18 runs, with either guidance and both models, sometimes leaving the
  placeholders in place.

The earlier pilot used only functional fixes, so I replayed a
documentation-only change, #22842 (a `juju show-unit` note in the 3.6 to
4.0 upgrade guide), with claude-sonnet-5.5 and gpt-6.1-terra, scored by
reading. The final row has two runs per model, before and after the Links
change, with the same QA in each pair; the other rows have one run each:

| QA wording                                                  | Claude QA                                     | GPT QA                                                                  |
| ----------------------------------------------------------- | --------------------------------------------- | ----------------------------------------------------------------------- |
| Previous                                                    | Read page; `juju` commands checking the claim | Read page                                                               |
| Intermediate: "no functional path, e.g. documentation-only" | "No functional path"; read page               | "No functional path"; read page                                         |
| Final                                                       | "No functional path"; read page               | Docs build (`make serve`, `make -C docs html`); `juju` commands on 3.6 and 4.0 |

The intermediate wording stopped agents from inventing setup, but it also
removed the one step that checked the documented behavior. The final
wording asks for a docs build and a behavior check instead. With it, GPT
gave a correct docs build and behavior check in both runs; Claude still
only read the page. No run listed unit tests. Before the Links change, two
Claude runs left `Fixes #19267` and `JUJU-` struck through; after it, both
models wrote `None` for the issue and Jira card.

