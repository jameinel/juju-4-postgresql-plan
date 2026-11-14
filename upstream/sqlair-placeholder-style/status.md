# sqlair-placeholder-style

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Shared findings](../../investigations/sqlair-placeholders.md)

## Current state and next actions

Planned 2026-11-14. Unclaimed; this directory is a task identifier, not
evidence that a branch or worktree exists. Target repository:
[canonical/sqlair](https://github.com/canonical/sqlair), base `main` at
`bd54c49` when planned. Alternative to
[`sqlair-dollar-placeholders`](../sqlair-dollar-placeholders/status.md);
both are prepared and proposed together, and the maintainers choose one.

Next actions:

- [ ] Before coding, confirm the API shape and names with the user
  (Open decisions).
- [ ] Reserve in the dashboard; record owner/session here.
- [ ] Create worktree `canonical/sqlair/.worktrees/placeholder-style` on
  branch `placeholder-style` from current `origin/main`; record the base.
- [ ] Implement the commits below, then validate.
- [ ] Verify the `$N` style on real dqlite (shared requirements).
- [ ] Prepare `pr.md` locally; publication, fork creation and pushes need
  the user's approval.

## Scope

Add a per-`DB` placeholder style with four variants, keeping today's
output as the default. Fixes
[sqlair#189](https://github.com/canonical/sqlair/issues/189) for drivers
that need `$N`, and makes MySQL (`?`) and Oracle (`:name`) usable. This is
the extensible option.

Out of scope: database-named dialects, driver auto-detection, dollar-quoted
or `E''` lexing, empty `IN ()`
([`sqlair-empty-in`](../sqlair-empty-in/status.md)).

## Design

Styles; the argument list is always built in first-appearance order:

| Style | SQL | Arguments | Reused parameter |
|---|---|---|---|
| `@name` (default) | `@sqlair_<id>` | `sql.Named("sqlair_<id>", v)` | same name; byte-identical to today |
| `:name` | `:sqlair_<id>` | `sql.Named("sqlair_<id>", v)` | same name |
| `$N` | `$<position>` | positional | same `$<position>` |
| `?` | `?` | positional | value appended again |

`$N` positions follow first appearance, so they bind correctly on SQLite
and PostgreSQL (shared findings).

1. Public API: a `PlaceholderStyle` type whose zero value is the `@name`
   style, selected per `DB` on `NewDB`; `TX` uses its `DB`'s style. No
   auto-detection: `*sql.DB` does not expose the driver name.
2. Internals: thread the style from `DB` / `TX` into
   `TypeBoundExpr.BindInputs` and `queryBuilder`. Replace the hard-coded
   `"@sqlair_"` sites with style methods that render a placeholder for a
   parameter (id, first-appearance position) and decide whether a reuse
   appends its value again. Keep the `@name` path producing identical SQL
   and arguments, so existing expectations stay unchanged. `Statement`
   stays style-independent; the statement cache already keys on
   (Statement, DB).
3. Parser: reject bypass `$<digits>` at `Prepare` in every style (shared
   commit). For the `?` style, bypass `?` outside literals and comments
   would shift arguments: have the parser record whether bypass SQL
   contains `?`, and fail at bind time in the `?` style with a clear
   error. The `?` stays valid for other styles (PostgreSQL JSON
   operator).

## Open decisions (confirm with the user before coding)

- API shape. Candidates:
  - `NewDB(db, WithPlaceholderStyle(Dollar))` (variadic options);
  - one option per style, for example `WithPlaceholderDollarNumbered()`
    (user suggestion);
  - an options struct, matching the `TXOptions` precedent:
    `NewDBWithOptions(db, &DBOptions{PlaceholderStyle: Dollar})`.
  Making `NewDB` variadic is source-compatible except where `NewDB` is
  used as a function value.
- Value names. User leaning (2026-11-14): `AtNamed`, `Dollar`,
  `Question`, `Colon`. Consider consistency (`ColonNamed`, or
  `DollarNumbered`) and the naming hazards in the shared API audit.

## Commits

Reuse the shared commits from
[`sqlair-dollar-placeholders`](../sqlair-dollar-placeholders/status.md)
(parser rejection; PostgreSQL driver-suite harness and CI). Whichever
session finishes them first records the hashes in both records.

1. `feat(parser): reject positional $N parameters` (shared).
2. `refactor(expr): route placeholders through a style`: internal style
   with only `@name`; no output change; existing tests unchanged.
3. `feat: add placeholder styles`: the public API, the `:name`, `$N` and
   `?` styles, the `?` bypass check, and table-driven `internal/expr`
   cases for each style (bulk insert mixing bulk and single-value
   columns, slices, omitted columns, literals; SQL text and arguments).
4. `test: run the driver suite against PostgreSQL` (shared harness), plus
   running the SQLite driver suite once per style (mattn accepts all
   four); PostgreSQL uses `$N`.
5. `docs: explain placeholder styles`: `docs/howto/db.md` section on
   choosing a style with a driver table, and a new record in
   `docs/decisions/` (next number after the backfill) with an index row.
   The record covers syntax-named styles rather than database dialects;
   why the setting belongs to the driver; the first-appearance
   invariant; why raw `$N` and, in the `?` style, raw `?` are rejected;
   and the always-`$N` alternative it was weighed against. Status:
   Proposed until merged.

The two shared decision-log commits from
[`sqlair-decision-records`](../sqlair-decision-records/status.md) come
before commit 1.

## Validation

- `go test ./...` with no PostgreSQL URL (SQLite suite, all styles).
- `SQLAIR_POSTGRESQL_TEST_URL=... go test ./...` against PostgreSQL 16
  through pgx `stdlib` with the `$N` style.
- Real dqlite through Juju with the default and `$N` styles.
- `go vet ./...`; build the docs if the docs toolchain is available.

## Acceptance criteria

- Default style: generated SQL and arguments byte-identical; existing
  tests pass unchanged.
- All four styles pass the SQLite driver suite; `$N` passes on
  PostgreSQL without SQL rewriting.
- In the `?` style, a reused single-value column in a bulk insert binds
  correctly (value repeated), and bypass `?` fails with a clear error.
- `$1` in bypass SQL fails at `Prepare` with a clear error.
- The PR description links the alternative PR and states that the
  default is unchanged.
