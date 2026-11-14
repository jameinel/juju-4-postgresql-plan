# sqlair-dollar-placeholders

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Owner: session `7bbea3b5-54ae-4766-8d29-6c2f341eda69`, authorized takeover
from `020f1b3f-f7b5-4bd4-9ad2-e27661961d66` on 2026-11-14.

This is the primary proposal for
[sqlair#189](https://github.com/canonical/sqlair/issues/189): one PR that
always emits numbered `$N` placeholders. The alternative,
[`sqlair-placeholder-style`](../sqlair-placeholder-style/status.md), is
linked from the PR and the issue as a branch, not opened as a second PR.

| | Value (observed 2026-11-14) |
|---|---|
| Repository | [canonical/sqlair](https://github.com/canonical/sqlair) |
| Worktree | `canonical/sqlair/.worktrees/numbered-placeholders` |
| Branch | `numbered-placeholders` (local, unpushed) |
| Base | `main` at `bd54c49` |
| Head | `2760dcf` |
| PR | None. Draft: [pr.md](pr.md) |
| Issue comment | None. Draft: [reply-issue-189.md](reply-issue-189.md) |

Commits, reviewable one at a time:

1. `ead635c refactor(expr): number inputs in order of first appearance`
   (shared)
2. `50117a0 feat(parser): reject raw numbered placeholders` (shared)
3. `eb8381a test: run SQLair against PostgreSQL in a separate module`
   (shared)
4. `2760dcf fix: emit numbered $N placeholders`

CONTRIBUTING.md says PRs are squash-merged with the PR description as the
commit message; [pr.md](pr.md) is written for that.

Next actions:

- [ ] Validate on real dqlite: build Juju against this branch with a
  `replace` directive and run domain state tests that include bulk
  inserts.
- [ ] With approval, push `numbered-placeholders` and
  `placeholder-style-option` to `jameinel/sqlair`, open the PR from
  [pr.md](pr.md), then post [reply-issue-189.md](reply-issue-189.md) with
  the PR number.
- [ ] With separate approval, delete the superseded fork branches listed
  below.

## Decisions (user, 2026-11-14)

- Start with the minimal change that is easy to review and understand;
  keep the configurable alternative available for comparison.
- Both branches share their first three commits, so that
  `numbered-placeholders...placeholder-style-option` shows only the cost of
  configurability.
- The root module stays on Go 1.24 without pgx. PostgreSQL tests live in
  the separate `tests/postgresql` module. The Go 1.26 bump is deferred.
- Unquoted `col$1` is rejected; the documented escape is `"col$1"`.
- Rationale goes in the PR description and the #189 comment. A decision
  log ([`sqlair-decision-records`](../sqlair-decision-records/status.md))
  and agent guidance
  ([`sqlair-agent-guidance`](../sqlair-agent-guidance/status.md)) are
  independent proposals.

## Compatibility

- Generated SQL and the argument form change for every driver; the public
  API does not.
- go-mssqldb accepts `@name` and `@pN`, not `$N`, so SQL Server stops
  working. MySQL did not work before and still does not.
- Raw `$N` now fails at `Prepare`. On `main` it fails at execution; with
  `$N` emission and no check, it would silently bind SQLair's first input.

## Validation (2026-11-14, Go 1.27.4, head `2760dcf`)

- `gofmt -l .` is empty; `go vet ./...` and `go test ./...` pass in the
  root and `tests/postgresql` modules; `go mod tidy -diff` is clean in
  both. The root `go.mod` is unchanged.
- PostgreSQL: `TestOutputs` and `TestInputs` pass. Before commit 4,
  `TestInputs` fails with `column "sqlair_0" does not exist`. No
  `sqlair_test_` schemas are left afterward.
- The functional QA in [pr.md](pr.md) printed the expected output on
  SQLite and PostgreSQL, and the PostgreSQL error before commit 4.
- The infrastructure commit's leak check fires and its `CASCADE` fallback
  cleans up. govulncheck is clean for the root and the submodule, with
  and without `-test`. actionlint flags only the existing old action pins.
- Docs build; `make spelling` flags only the existing `security.md` words
  (`Cryptographic`, `TLS`, `utilizing`).
- Not run: real dqlite; the documented Docker command (no local Docker).

## Superseded branches

Reference only; do not extend them.

| Branch | Head | Location |
|---|---|---|
| `dollar-placeholders` | `2156df4` | local and `jameinel/sqlair` |
| `dollar-placeholders-on-postgresql` | `1213a73` | local only |
| `postgresql-test-infrastructure` | `ccf6072` | local and `jameinel/sqlair` |

The previous, longer record is in planning commit `09dc917`.

## Out of scope

Empty `IN ()` rendering
([`sqlair-empty-in`](../sqlair-empty-in/status.md)), dollar-quoted string
lexing, and bumping the old `actions/checkout`/`actions/setup-go` pins.
