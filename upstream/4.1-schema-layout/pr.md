# Local PR draft

Destination: `juju/juju`, base `4.1`, head `jameinel:4.1-schema-layout`.
Branch `9da8100e16` was pushed to the fork with explicit approval on
2026-11-12. Action after separate explicit approval: create the PR.
No PR has been published.

Title: `refactor(schema): move dqlite ddl under domain/schema/dqlite`

---

Move controller and model SQL patches, trigger packages and historical release
DDL under `domain/schema/dqlite`. Keep schema assembly APIs, patch ordering,
hash inputs and all relocated file contents unchanged. Update imports, embeds,
generation destinations, patch discovery, workflow paths and documentation.

Make the existing dialect explicit so a PostgreSQL schema can sit beside it
without treating schema correctness as a testing-only concern. Keep this
mechanical prerequisite independent of SQL conversion, PostgreSQL output,
driver support, backend selection and shared SQL fixes.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [x] `doc.go` added or updated in changed packages

## QA steps

Check generation writes only to the relocated asset tree. With the normal Go
build prerequisites installed, run from the repository root:

```sh
go generate ./domain/schema/...
git diff --exit-code -- domain/schema/dqlite
test ! -d domain/schema/controller
test ! -d domain/schema/model
```

Expect no generated-content changes and no recreated old directories.

To exercise controller and model schema creation through the product, use a
working local LXD cloud and a dedicated disposable controller. Build from the
reviewed checkout:

```sh
make install
export PATH="$(go env GOPATH)/bin:$PATH"
juju bootstrap localhost schema-layout-qa --build-agent
juju add-model schema-layout-model
juju model-config logging-config='<root>=INFO'
juju model-config logging-config
juju status
```

Expect bootstrap and model creation to succeed, `<root>=INFO` to be returned,
and status to show the new model without missing-table or trigger errors.
Clean up the dedicated QA controller:

```sh
juju destroy-controller schema-layout-qa --destroy-all-models \
    --destroy-storage -y
```

## Documentation changes

Update schema package documentation, schema agent guidance and path references
for the relocation. No CLI/API workflow changes.

## Links

~Issue: no issue reference.~

~Jira card: not applicable.~
