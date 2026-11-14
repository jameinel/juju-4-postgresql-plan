# sqlair-placeholder-style

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Owner: session `7bbea3b5-54ae-4766-8d29-6c2f341eda69`, authorized takeover
from `6e9894b2-d722-418e-9dac-32096188585b` on 2026-11-14.

This is the alternative to
[`sqlair-dollar-placeholders`](../sqlair-dollar-placeholders/status.md)
for [sqlair#189](https://github.com/canonical/sqlair/issues/189). It keeps
`@sqlair_N` as the default and adds a per-database placeholder style
option. It is published as a branch and linked from the primary PR and the
issue; no PR is planned unless the maintainers prefer it.

| | Value (observed 2026-11-14) |
|---|---|
| Repository | [canonical/sqlair](https://github.com/canonical/sqlair) |
| Worktree | `canonical/sqlair/.worktrees/placeholder-style-option` |
| Branch | `placeholder-style-option` (pushed to `jameinel/sqlair`) |
| Parent | `eb8381a`, the third shared commit of `numbered-placeholders` |
| Head | `60c75b0` locally and on `jameinel/sqlair` |
| PR | None planned. Description if chosen: [pr.md](pr.md) |

The commit adds 268 lines and removes 6 across 7 files. The dollar-only
commit adds 122 and removes 64, mostly renumbered test expectations.

Next actions:

- [x] Push the approved head alongside `numbered-placeholders`.
- [x] Commit PostgreSQL test and documentation link follow-ups.
- [x] Push follow-up changes to `jameinel/sqlair`.
- [ ] If the maintainers choose this approach, prepare a PR from
  [pr.md](pr.md) for approval.

## Design

- `internal/expr`: `PlaceholderStyle` (`AtNamed`, `Dollar`, `Colon`);
  `queryBuilder.addParam` switches on it. `BindInputsWithStyle` is new;
  `BindInputs` uses `AtNamed`.
- Public API: `PlaceholderStyle` with `AtNamedPlaceholders` (zero value),
  `DollarPlaceholders` and `ColonPlaceholders`; `DBOptions`;
  `NewDBWithOptions(sqldb, *DBOptions) (*DB, error)`, which returns an
  error for a nil `sql.DB` or an unknown style. `NewDB` is unchanged.
  `DB.Query` and `TX.Query` use the database's style.
- Tests: expr output for each style; SQLite bulk insert, transaction and
  slice query for each style; `NewDBWithOptions` errors; the PostgreSQL
  `TestInputs` with `DollarPlaceholders`.
- Docs: `docs/howto/db.md` "Choose a placeholder style".

## Decisions (user, 2026-11-14)

- Syntax-named styles, not database names; see the API audit in the
  [shared findings](../../investigations/sqlair-placeholders.md#api-audit).
- No `?` style: it cannot reuse a value, so SQLair would need to repeat
  values per use, track raw `?` and check at bind time, and MySQL is
  untested. The docs say so.
- Do not claim Oracle support for `ColonPlaceholders`; it is tested only
  with SQLite.
- Errors, not panics, for invalid options.
- Match the primary proposal's lightweight PostgreSQL syntax rejection and
  minimum/latest Go CI matrix; document the SQLite identifier-quoting
  limitation rather than expanding the parser.

## Validation (2026-11-14, Go 1.27.4, head `1fe3704`)

- `gofmt -l .` is empty; `go vet ./...` and `go test ./...` pass in both
  modules; the new gocheck tests ran and passed; `go mod tidy -diff` is
  clean in both.
- PostgreSQL `TestOutputs` and `TestInputs` pass.
- The functional QA in [pr.md](pr.md) printed the expected output for
  every style on SQLite and for `dollar` on PostgreSQL; `at` on PostgreSQL
  fails with `column "sqlair_0" does not exist`.
- `make spelling` flags only the existing `security.md` words.
- Real dqlite: Juju 4.1 state constraint tests pass with a session-local
  modfile replacing SQLair with this worktree and `make run-go-tests`.
- Fork CI: Go tests (including PostgreSQL) and vulnerability scan passed.
  Documentation spelling failed on the existing `security.md` words, and
  link checking failed on API anchors not yet published on pkg.go.dev.
- The committed follow-up changes raise the PostgreSQL module to Go 1.26,
  use generic PostgreSQL URL instructions in CONTRIBUTING and link to the
  package page rather than unreleased API anchors. Root and PostgreSQL vet
  and tests, including live PostgreSQL with Go 1.26.4, pass; `make linkcheck`
  passes.
- Repeated root and PostgreSQL tests, vet, module tidy checks and
  govulncheck with tests; all pass. The changed workflows use current action
  releases pinned by SHA, and actionlint passes for those workflows.
- The PR draft has the same Juju setup and verification limits as the
  [primary draft](../sqlair-dollar-placeholders/status.md).
- On head `60c75b0`, root formatting, vet, tests and module tidy checks pass.
  PostgreSQL vet and live tests pass on Go 1.26.0 and 1.27.4, including the
  ambiguous-syntax regressions; its module tidy check is clean. Actionlint
  passes for both changed workflows. Documentation builds; spelling reports
  only the existing `security.md` words.
- Real-dqlite Juju constraint tests pass again using `make run-go-tests` and a
  session-local modfile pointing at this head. SQLite identifier quoting is
  not parsed and can hide raw placeholders, as documented in the draft and
  reference documentation.

## Superseded branches

Reference only; do not extend them. Their worktrees have been removed, but
the branches remain.

| Branch | Head | Location |
|---|---|---|
| `placeholder-style` | `cfafcde` | local and `jameinel/sqlair` |
| `placeholder-styles` | `69db9bc` | local and `jameinel/sqlair` |
| `postgresql-test-infrastructure` | `ccf6072` | local and `jameinel/sqlair` |

`placeholder-styles` also has the Go 1.26 bump, dependency updates and an
`AGENTS.md`; those are deferred or tracked separately. The previous,
longer record, with CI links for those heads, is in planning commit
`09dc917`.
