# Local PR draft

Destination: `juju/juju`, base `4.1`, head `jameinel:4.1-schema-layout`,
draft PR. The local and fork heads are `9c8f98cd16`, based on `b622931495`.
PR creation requires separate explicit approval. No PR has been published.

Title: `refactor(schema): order and relocate schema definitions`

---

Order controller and model schema definitions so foreign-key targets and view
sources exist before their consumers. Split out shared prerequisites where
whole-file ordering cannot satisfy the dependencies. Give each base patch a
distinct numeric position, spaced by ten, to leave room for future insertions
without renumbering later files, where schema evolution permits. Add a guard
for table keys, view dependencies and trigger dependencies.

Move the ordered SQL patches and trigger packages under
`domain/schema/dqlite`, and relocate generated release-DDL output paths.
Update imports, embeds, generation destinations, patch discovery, workflow
paths and documentation. The move itself preserves patch ordering, hash
inputs and relocated file contents.

The assembled controller and model patch statements, including triggers,
remain identical to #23890 in content and order. The layout move does not
change the patch hash chains.

Make the existing dialect explicit so a PostgreSQL schema can sit beside it
without treating schema correctness as a testing-only concern. Keep this
mechanical prerequisite independent of SQL conversion, PostgreSQL output,
driver support, backend selection and shared SQL fixes. The dqlite schema
is also used by the SQLite test backend.

The ordering changes the 4.1 beta base-schema hash chain. Earlier 4.1 beta
databases cannot upgrade in place across the change. Released 4.0 models
continue to reach 4.1 through migration; historical release DDL is unchanged.

This draft is built on #23890: it includes that PR's ordering commit and a
separate layout commit. The intended sequence is to land #23890 first, then
review and land the layout follow-up. Until then, this PR targets `4.1` and
shows the combined diff for comparison. If #23890 is squash-merged, this
branch may need rebasing onto the resulting commit.

Would reviewers prefer the separate ordering and layout PRs, or a single
combined PR so the files are renamed once in the final diff?

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [x] `doc.go` added or updated in changed packages

## QA steps

From a checkout containing both commits, compare the controller and model
asset trees with the ordering commit:

```sh
set -e
ORDER=b622931495f0fb3a2242418c82e2fcb152183354
LAYOUT=9c8f98cd16289cb0e17b9be9ac587d59a683b285
test "$(git --no-pager rev-parse "$ORDER:domain/schema/controller")" = \
     "$(git --no-pager rev-parse "$LAYOUT:domain/schema/dqlite/controller")"
test "$(git --no-pager rev-parse "$ORDER:domain/schema/model")" = \
     "$(git --no-pager rev-parse "$LAYOUT:domain/schema/dqlite/model")"
git --no-pager diff --exit-code "$ORDER" "$LAYOUT" -- core/database/schema
git --no-pager diff "$ORDER" "$LAYOUT" -- \
    domain/schema/controller.go domain/schema/model.go
```

Expect the tree and hash-implementation comparisons to succeed without
output. The trees include filenames, SQL contents and trigger sources.
Inspect the final diff and confirm schema assembly changes only asset paths.

With the normal Go build prerequisites installed, regenerate triggers and the
exported schema from a clean layout checkout:

```sh
go generate ./domain/schema/...
make rebuild-export
git --no-pager diff --exit-code
test ! -d domain/schema/controller
test ! -d domain/schema/model
```

Expect no generated-content changes and no recreated old directories.
`make rebuild-export` applies the schema to generate export code; it does not
by itself compare patch hashes. For a live database comparison, use the QA
steps in #23890.

## Documentation changes

Update schema package documentation, schema agent guidance and path references
for the relocation. No CLI/API workflow changes.

## Links

Prerequisite ordering PR: #23890. Its QA steps also describe a
before-and-after comparison of the controller and model database schemas.

~Issue: no issue reference.~

~Jira card: not applicable.~
