---
name: prepare-pr-review-response
description: Evaluate pull request review feedback with the user, prioritizing human reviewers over Van Cleef and other bots, then prepare a focused patch, an updated prospective PR description, and a complete review response for human approval. Use when asked to respond to or address a PR review.
---

# Prepare a PR review response

The planning repository is three directories above this skill's directory.
Read its AGENTS.md and the selected `upstream/<branch>/status.md`. The
session may start in its parent, alongside multiple Juju worktrees. Work
on the existing PR branch in its authorized worktree; do not create a
replacement branch or disturb another owner's checkout. Read the target
repository's applicable instructions and PR template.

## Understand the review

1. Identify the PR and the review or feedback the user wants addressed.
   If the target or intended scope is ambiguous, ask. Inspect the live PR
   description, base, current head, commit history, checks, review bodies,
   issue comments, and inline threads, including replies, resolution and
   outdated status. Follow pagination. Prefer `gh` for GitHub operations.
2. Read the relevant code and tests, and compare each review point with later
   commits and author responses. Review bodies may contain feedback even
   without inline threads. Do not treat resolved, outdated, or bot-generated
   feedback as automatically correct, incorrect, or already answered.
3. Make an item-by-item working inventory of the requested feedback, including
   warnings, suggestions, questions and distinct subpoints. Record whether
   each is already addressed, needs a change or explanation, should be
   deferred, or warrants disagreement. Base decisions on the code and
   evidence; do not invent a follow-up branch or claim a fix is complete.
   Ask the user when behavior, scope, ownership or a meaningful tradeoff
   requires a decision. If the branch or worktree is dirty or owned by
   another session, preserve its changes and obtain authorization before
   touching it.

## Weigh reviewers

- Human reviewers come first. Their comments, questions and approval
  decide whether the PR merges, so answer every human point directly and
  list human feedback first in the inventory and the discussion with the user.
- Van Cleef (`lee-van-cleef[bot]`) is an important automated reviewer. Its
  review body opens with a "Code Review Summary" and a verdict that counts
  issues and suggestions; it may also leave inline threads. Inventory each
  issue and suggestion and answer all of them, but treat them as findings to
  verify, not instructions. Its approval or verdict is not a human approval.
- Other bots (for example linters or CI commenters) are lower priority;
  address them when relevant to correctness or merge requirements.
- When a human and a bot disagree, favor the human's direction, but tell the
  user about the conflict and any evidence that the bot's point still holds.

## Discuss the feedback before changing code

Do not assume a requested change is correct because a reviewer asked for it,
whether the reviewer is a human or a bot. Before implementing, present the
working inventory to the user and discuss it. For each item give:

- **Request:** who asked, what they asked, and a link to the comment.
- **Context:** the relevant code, behavior, history or plan decision, so the
  user can judge the point without opening the PR.
- **Assessment:** whether the point is correct, partly correct, already
  addressed or mistaken, and why. Explain what the change would improve and
  what it would cost or risk: behavior changes, scope growth, conflicts with
  the plan's decisions or root-cause contract, test burden, or divergence
  from upstream conventions.
- **Recommendation:** change, explain, defer or push back, and the proposed
  shape of any change. State uncertainty and alternatives where they matter.

Wait for the user's agreement before implementing or drafting a final
response for any item that changes behavior, widens scope, involves
disagreement with a reviewer, or has a meaningful tradeoff. Items that are
clearly correct and mechanical (a typo, a missing test for stated behavior)
may be grouped and proposed together, but still list them. If the user
disagrees with an assessment, update the inventory and follow their decision.

## Implement and validate

- Make focused fixes on the existing branch for the agreed items. Keep
  unrelated concerns separate, preserve existing behavior unless a change is
  agreed, and add regression tests for changed behavior. Disagreement or
  deferral is acceptable when supported by a specific rationale.
- Run the checks appropriate to the change, following the target repository
  and task guidance. For the stock upstream Juju plan, consult the
  [breakout validation guide](../upstream-breakout/references/validation.md):
  check tool availability and use the compiled `txncheck`, adding
  `$HOME/go/bin` to `PATH` when needed. Report actual results and checks not
  run; skipped checks are not passes.
- Inspect the resulting diff and current branch, then create a conventional
  follow-up commit with the required co-author trailer when a patch is needed.
  Do not amend published commits unless explicitly requested. Do not commit
  unrelated changes. If no code change is appropriate, say so.

## Prepare separate review artifacts

Prepare both artifacts locally, even when only one needs publication. Clearly
identify when the existing description needs no change.
Use `upstream/<branch>/pr.md` for the description and a sibling
`reply-<reviewer-or-topic>.md` for each response. Add a date/suffix when
needed, not a replies subdirectory. Existing saved drafts may be historical;
compare them with the current published text before editing.

**PR description:** Present the final proposed change as a prospective merge
commit message. Explain what it does, why, important behavior and tradeoffs,
and reviewer context needed to understand it. Preserve important existing
context, and incorporate new facts or clarifications discovered in review
whenever they change how the final patch should be understood, even if the
code itself did not change. Include an alternative or earlier approach when
it helps explain the final design or its tradeoffs; omit iteration details
that only recount the review process. Use the repository's PR template and
give explicit, reproducible functional QA steps, including setup commands
and expected results. Update QA when the final behavior requires it. Review
chronology and completed-check history belong in the review response.

**Review response:** Address every item in the selected review, including
distinct subpoints. For each, say what changed, what already satisfies the
request, where it is deferred, or why you disagree; include evidence and links
when useful. Do not say a future change or check has happened. Mention relevant
validation and limitations accurately, without presenting a timed-out stress
run or skipped check as a complete pass. Keep the response direct and
respectful, and avoid repeating the whole PR description.

Reply to human reviewers where they commented, answering their questions
directly. Answer Van Cleef's issues and suggestions as well, either in its
threads or grouped in one reply to its review summary; keep those replies
brief and evidence-based. Where you push back, give the reason agreed with the
user rather than a bare refusal.

Before handing off, cross-check the working inventory against the response.
If an item is not answered or depends on a user decision, identify it
explicitly rather than omitting it. Keep locally staged text near 80 columns
where practical without harming readability.

## Review and publication gates

- Show the user the follow-up commit and exact push destination before
  pushing. Verify the current PR head and remote branch first; stop on
  unexpected divergence rather than overwriting another person's work.
- Prepare and show the exact proposed PR body, destination, and edit action,
  and the exact review response, destination, and comment or reply action.
  Obtain explicit human approval for each publication action and its exact
  content before publishing. A request to address feedback, tool permission,
  approval of a code patch, or a push is not approval to publish text. If
  content or destination changes after approval, request approval again.
- For an approved PR body edit, use `gh api -X PATCH` on
  `repos/OWNER/REPO/issues/NUMBER` with `-F body=@FILE` directly; do not use
  `gh pr edit`. Verify the published body afterward.
- Do not publish, edit or delete existing GitHub content, or resolve review
  threads without explicit approval for that specific action. After an
  approved publication, verify the resulting GitHub content and link.
- Summarize the branch and commit, validation and limitations, prepared or
  published artifacts, and any remaining decisions or review actions.
- Update only the branch's `status.md` and assigned artifacts with the
  dated head, review disposition, evidence and next actions. Notify the
  coordinator of changed scope, new findings or cross-branch effects; shared
  dashboards and PoC stage progress are not routine review-worker edits.
  Retain ownership while blocked. Follow AGENTS.md's shared Git rules:
  coordinate index/commit operations and stage only explicitly named files,
  preserving other sessions' dirty or staged work.
