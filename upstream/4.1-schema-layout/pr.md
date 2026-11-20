# Local PR draft

Published draft: https://github.com/juju/juju/pull/23907
Base: `4.1`; head: `jameinel:4.1-schema-layout` at `9c8f98cd16`, based on
`b622931495`. The body below matches the published description.

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
assets with the ordering commit. Create temporary checkouts:

```sh
ORDER=b622931495f0fb3a2242418c82e2fcb152183354
LAYOUT=9c8f98cd16289cb0e17b9be9ac587d59a683b285
QA_ROOT="$(mktemp -d)"
git worktree add --detach "$QA_ROOT/order" "$ORDER"
git worktree add --detach "$QA_ROOT/layout" "$LAYOUT"
```

Compare filenames and contents, and check the hash code:

```sh
diff -ru "$QA_ROOT/order/domain/schema/controller" \
    "$QA_ROOT/layout/domain/schema/dqlite/controller"
diff -ru "$QA_ROOT/order/domain/schema/model" \
    "$QA_ROOT/layout/domain/schema/dqlite/model"
git diff "$ORDER" "$LAYOUT" -- core/database/schema
```

Expect all three comparisons to produce no output. The directory diffs show
any missing or changed SQL files and trigger sources.

Inspect the schema assembly changes separately:

```sh
git diff "$ORDER" "$LAYOUT" -- \
    domain/schema/controller.go domain/schema/model.go
```

Expect a diff that changes only asset paths. Remove the temporary checkouts:

```sh
git worktree remove "$QA_ROOT/order"
git worktree remove "$QA_ROOT/layout"
rmdir "$QA_ROOT"
```

With the normal Go build prerequisites installed, regenerate triggers and the
exported schema from a clean layout checkout:

```sh
go generate ./domain/schema/...
make rebuild-export
git diff --exit-code
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
