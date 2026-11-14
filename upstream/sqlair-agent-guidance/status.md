# sqlair-agent-guidance

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md)

## Current state and next actions

Owner: session `7bbea3b5-54ae-4766-8d29-6c2f341eda69`, 2026-11-14.

Independent proposal for a root `AGENTS.md` in
[canonical/sqlair](https://github.com/canonical/sqlair). It does not depend
on either placeholder branch; its placeholder invariant avoids naming a
syntax so that it does not conflict with them.

| | Value (observed 2026-11-14) |
|---|---|
| Worktree | `canonical/sqlair/.worktrees/agent-guidance` |
| Branch | `agent-guidance` (local, unpushed) |
| Base | `main` at `bd54c49` |
| Head | `bc156ff docs: add agent contributor guidance` |
| PR | None; no description drafted |

The file has 70 lines: code map and invariants, validation commands, test
conventions, docs tooling and commit conventions, deferring policy to
CONTRIBUTING.md. Its claims about the docs Makefile (`-W`, `woke`,
`linkcheck`), the `-check.f` command and CONTRIBUTING were checked against
`main`.

Next actions:

- [ ] Decide whether to propose it, and when, relative to the placeholder
  PR.
- [ ] Draft a PR description and obtain approval before publishing.

## Open questions for the maintainers

- Docs spelling is checked as `en_GB`, but some content uses US spelling
  (for example `utilizing` in `security.md`). Should one be standardized?
- Go files mix Apache 2.0 and AGPLv3 headers. The guidance says not to
  reconcile them silently; this needs a separate maintainer decision.

## Evidence

Of 143 commit subjects on `main`, 42 use conventional style; 27 of the 40
most recent PR titles do.
