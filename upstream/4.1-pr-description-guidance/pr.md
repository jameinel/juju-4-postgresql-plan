# docs: clarify pr description and review comment roles

Clarify the distinction between a pull request description and follow-up
review comments in the agent commit instructions. The PR description
becomes the future commit message and should describe the final change,
its rationale, and how to verify it. QA instructions may change when the
verification procedure changes. Reviewer responses, review chronology and
follow-up validation results belong in comments instead.

This guidance is independent of the integer-schema changes and does not
change Juju behaviour or the required PR-template structure.

## Checklist

- [ ] ~Code style: imports ordered, good names, simple structure, etc~
- [ ] ~Comments saying why design decisions were made~
- [ ] ~Go unit tests, with comments saying what you're testing~
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

1. Review the **Pull Request Description** section of
   `.github/instructions/agent-commit.instructions.md`.
2. Confirm that a changed verification procedure belongs in the PR's QA
   instructions, while reviewer replies and follow-up run results belong
   in comments.
3. Confirm that the instruction still requires the full repository PR
   template and does not change commit-message conventions.

No executable code changes; no automated tests are required.

## Documentation changes

Add the description/comment distinction to the existing agent commit
instructions. No user-facing documentation changes.

## Links

**Issue:** N/A

**Jira card:** N/A
