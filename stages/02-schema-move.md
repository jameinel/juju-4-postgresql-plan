# Stage 02: move schema to `domain/schema/dqlite`

Branch `pg-poc/02-schema-move`, on `pg-poc/01-provision`.

- Goal: separate the dqlite DDL so a PostgreSQL schema can sit beside it.
  Mechanical, with no behaviour change.
- Steps:
  1. `git mv domain/schema/{controller,model}` (SQL, triggers, release
     DDL) under `domain/schema/dqlite/`, keeping the existing Go API
     (`schema.ControllerDDL()`, `schema.ModelDDL()`) as thin wrappers.
  2. Update `go:generate` directives, `generate/triggergen`, embed paths and
     `domain/schema/AGENTS.md`.
- Verification: `go generate ./domain/schema/...` produces no diff;
  `go test ./domain/schema/... ./core/database/...` passes; schema hashes are
  unchanged.
- Status: in review. Committed on `pg-poc/02-schema-move` (one `refactor`
  commit); verification passed.
- Restart notes:
  - Done 2026-10-23: `git mv` of `domain/schema/{controller,model}` to
    `domain/schema/dqlite/`; embed paths, `go:generate` destinations,
    trigger package imports, `generate/ddlgen`, `.github/workflows/ddl.yml`
    and `context-tests.yml`, comments and `domain/schema/AGENTS.md` updated.
    The Go package stays `domain/schema` with `ControllerDDL()` and
    `ModelDDL()` unchanged, so no callers changed.
  - `go generate ./domain/schema/...` left no diff; `go test
    ./domain/schema/... ./core/database/... ./domain/agentbinary/...
    ./domain/modelmigration/...` passes (the schema suites cover hashes).
  - Next: user review.
- Findings (for later stages):
  - Stage 4 should add `domain/schema/postgresql/` beside `dqlite/`; the
    `schema` package (`controller.go`, `model.go`, `patch.go`) still owns
    the dqlite embed and `readPatches`, and will need a backend selector
    (resolved by D11).
  - Historic `.ddl` release files contain stale path comments; left as is.
  - The build errors and `gofmt` noise seen during this stage were a Go
    1.24 `gofmt` on `PATH` plus ungofmt-ed generated trigger files; both
    are resolved by [Stage 00](00-base-fixes.md).
