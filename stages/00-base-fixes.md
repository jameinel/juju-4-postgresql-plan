# Stage 00: base fixes

Branch `pg-poc/00-base-fixes`, the bottom of the stack, on upstream `4.1`
at `a8591fe1b8`. It fixes pre-existing problems unrelated to the PoC.
Never commit stage work to it.

- Status: done, in review.

- `go.mod` pins `go 1.27.2`, and `domain/removal/service` uses generic
  methods, which need Go 1.27. `go build ./...` passes. The "method must
  have no type parameters" errors (seen in several stage sessions) came
  from an older `gofmt` on `PATH` (snap Go 1.24), not from the build. Use
  the Go 1.27 `gofmt`.
- With Go 1.27, `gofmt -l` listed only the generated trigger files; the
  other files were Go 1.24 `gofmt` artefacts. Fixed: `generate/triggergen`
  formats its output with `go/format`, and the trigger files are
  regenerated. `go generate ./domain/schema/...` must leave no diff.
- `go vet ./generate/...` flags a json tag on an unexported field in
  `generate/schemagen/jsonschema-gen/reflect_test.go`. It is intentional
  (the test checks such fields are ignored) and suppressed for
  golangci-lint with `//nolint:govet`, so it is left as is. Plain
  `go vet ./...` is not a gate in this repository.
- `make pre-check` runs static analysis only with `STATIC_ANALYSIS=1`. It
  needs golangci-lint 2.14 exactly, and `shellcheck` on `PATH`.
