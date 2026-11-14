# sqlair-dollar-placeholders

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Owner: session `7bbea3b5-54ae-4766-8d29-6c2f341eda69`, authorized takeover
from `020f1b3f-f7b5-4bd4-9ad2-e27661961d66` on 2026-11-14.

This is the primary proposal for
[sqlair#189](https://github.com/canonical/sqlair/issues/189): one PR that
always emits numbered `$N` placeholders. The alternative,
[`sqlair-placeholder-style`](../sqlair-placeholder-style/status.md), is
linked from the PR as a branch, not opened as a second PR. The issue reply
is prepared locally.

| | Value (observed 2026-11-14) |
|---|---|
| Repository | [canonical/sqlair](https://github.com/canonical/sqlair) |
| Worktree | `canonical/sqlair/.worktrees/numbered-placeholders` |
| Branch | `numbered-placeholders` (pushed to `jameinel/sqlair`) |
| Base | `main` at `bd54c49` |
| Head | `ae3d19e` locally and on `jameinel/sqlair` |
| PR | [#195](https://github.com/canonical/sqlair/pull/195), open; description: [pr.md](pr.md) |
| Issue comment | None. Draft: [reply-issue-189.md](reply-issue-189.md) |

Commits, reviewable one at a time:

1. `ead635c refactor(expr): number inputs in order of first appearance`
   (shared)
2. `50117a0 feat(parser): reject raw numbered placeholders` (shared)
3. `eb8381a test: run SQLair against PostgreSQL in a separate module`
   (shared)
4. `2760dcf fix: emit numbered $N placeholders`
5. `1a4d508 test: require Go 1.26 for PostgreSQL tests`
6. `66bf0d4 ci: pin current GitHub Actions releases`
7. `ae3d19e fix: reject ambiguous SQL placeholder syntax`

CONTRIBUTING.md says PRs are squash-merged with the PR description as the
commit message; [pr.md](pr.md) is written for that.

Next actions:

- [x] Push the approved branch heads to `jameinel/sqlair`.
- [x] Validate real dqlite with a Juju `replace` directive and state tests
  that include bulk inserts.
- [x] Commit the local PostgreSQL test follow-up.
- [x] Push the follow-up to `jameinel/sqlair`.
- [x] Open the approved PR and verify its title, head, base and description.
- [ ] Obtain approval before posting [reply-issue-189.md](reply-issue-189.md).
- [ ] With separate approval, delete the superseded fork branches listed
  below.

## Decisions (user, 2026-11-14)

- Start with the minimal change that is easy to review and understand;
  keep the configurable alternative available for comparison.
- Both branches share their first three commits, so that
  `numbered-placeholders...placeholder-style-option` shows only the cost of
  configurability.
- The root module stays on Go 1.24 without pgx. PostgreSQL tests live in
  the separate `tests/postgresql` module and require Go 1.26.
- Unquoted `col$1` is rejected; the documented escape is `"col$1"`.
- Rationale goes in the PR description and the #189 comment. A decision
  log ([`sqlair-decision-records`](../sqlair-decision-records/status.md))
  and agent guidance
  ([`sqlair-agent-guidance`](../sqlair-agent-guidance/status.md)) are
  independent proposals.
- The README's "Database agnostic" claim states an intent, not a guarantee of
  current driver support; it does not need changing for this proposal.
- Direct review: preserve the lightweight parser, reject the identified
  ambiguous PostgreSQL syntax, and test the PostgreSQL module with both its
  minimum and the latest Go version. Error precedence for invalid queries is
  not guaranteed.
- Document, rather than implement parsing for, SQLite bracket- and
  backtick-quoted identifiers. Quotes or comment markers inside them can
  hide raw placeholders; use double-quoted identifiers and SQLair inputs.

## Compatibility

- Generated SQL and the argument form change for every driver; the public
  API does not.
- go-mssqldb accepts `@name` and `@pN`, not `$N`, so SQL Server stops
  working. MySQL did not work before and still does not.
- Raw `$N` normally fails at `Prepare`. On `main` it fails at execution; with
  `$N` emission it can silently bind SQLair's first input when the check is
  confused by unsupported identifier quoting.
- Dollar-quote delimiters, nested block comments, and a backslash immediately
  before a single quote in a literal are rejected. This includes some valid
  SQL; dollar-quote-like text in unquoted identifiers can also be rejected.

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
  and without `-test`.
- Docs build; `make spelling` flags only the existing `security.md` words
  (`Cryptographic`, `TLS`, `utilizing`).
- Real dqlite: Juju 4.1 at `4262e78b24`, with a session-local modfile
  replacing SQLair with this worktree, passed
  `TestCreateApplicationWithUnits`, `TestSetConstraintFull` and
  `TestSetConstraintsReplacesPreviousZones` using `dqlite,libsqlite3` and
  Juju's `_deps` toolchain. `make run-go-tests` also passed seven constraint
  tests with the same modfile supplied through `GOFLAGS`.
- Fork CI: Go tests (including PostgreSQL) and vulnerability scan passed.
  Documentation spelling failed on the existing `security.md` words above.
- The committed follow-up changes raise the PostgreSQL module to Go 1.26
  and replace Docker instructions in CONTRIBUTING with a generic connection
  URL. Root and PostgreSQL vet and tests, including the live Juju PostgreSQL
  tests, pass with these changes. The PostgreSQL tests also pass with Go
  1.26.4.
- Repeated root and PostgreSQL tests, vet, module tidy checks and
  govulncheck with tests; all pass. Real-dqlite Juju state tests pass again.
- After actionlint reported the old action versions, the changed workflows
  were updated to current releases pinned by SHA. Actionlint passes for both
  changed workflows; the separate TIOBE workflow retains its existing pins
  and custom runner labels.
- The PR draft now describes a Juju setup. Readiness commands were checked
  against the existing model, and URL construction against synthetic data;
  fresh deployment and credential extraction were not run end to end.
- On head `ae3d19e`, root formatting, vet, tests and module tidy checks pass.
  PostgreSQL vet and live tests pass on Go 1.26.0 and 1.27.4, including the
  three ambiguous-syntax regressions; its module tidy check is clean.
  Actionlint passes for both changed workflows. The documentation builds;
  spelling reports only the pre-existing `security.md` words listed above.
- Real-dqlite Juju constraint tests pass again using `make run-go-tests` and a
  session-local modfile pointing at this head. SQLite probes confirmed the
  documented identifier-quoting limitation. No review response is required:
  the feedback was provided directly, not on a published PR.

## Superseded branches

Reference only; do not extend them. Their worktrees have been removed, but
the branches remain.

| Branch | Head | Location |
|---|---|---|
| `dollar-placeholders` | `2156df4` | local and `jameinel/sqlair` |
| `dollar-placeholders-on-postgresql` | `1213a73` | local only |
| `postgresql-test-infrastructure` | `ccf6072` | local and `jameinel/sqlair` |

The previous, longer record is in planning commit `09dc917`.

## Out of scope

Empty `IN ()` rendering
([`sqlair-empty-in`](../sqlair-empty-in/status.md)), dollar-quoted string
lexing, and updating the unrelated TIOBE workflow.
