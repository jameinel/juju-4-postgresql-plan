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

Compare the complete assembled patch sequences against the ordering commit.
From a checkout containing both commits, with Go installed:

```sh
set -e
export QA_ROOT="$(mktemp -d)"
git --no-pager worktree add --detach "$QA_ROOT/order" b622931495f0fb3a2242418c82e2fcb152183354
git --no-pager worktree add --detach "$QA_ROOT/layout" 9c8f98cd16289cb0e17b9be9ac587d59a683b285
cat > "$QA_ROOT/patches.go" <<'EOF'
package main

import (
	"encoding/json"
	"log"
	"os"

	coreschema "github.com/juju/juju/core/database/schema"
	"github.com/juju/juju/domain/schema"
)

func statements(s *coreschema.Schema) []string {
	patches := s.Patches()
	result := make([]string, len(patches))
	for i, patch := range patches {
		result[i] = coreschema.Stmt(patch)
	}
	return result
}

func main() {
	output := struct {
		Controller []string `json:"controller"`
		Model      []string `json:"model"`
	}{statements(schema.ControllerDDL()), statements(schema.ModelDDL())}
	if err := json.NewEncoder(os.Stdout).Encode(output); err != nil {
		log.Fatal(err)
	}
}
EOF
(cd "$QA_ROOT/order" && go run "$QA_ROOT/patches.go") > "$QA_ROOT/order.json"
(cd "$QA_ROOT/layout" && go run "$QA_ROOT/patches.go") > "$QA_ROOT/layout.json"
cmp "$QA_ROOT/order.json" "$QA_ROOT/layout.json"
git --no-pager diff --exit-code b622931495f0fb3a2242418c82e2fcb152183354 9c8f98cd16289cb0e17b9be9ac587d59a683b285 -- core/database/schema
```

Expect both comparisons to produce no output. The JSON arrays preserve patch
boundaries and order, including generated triggers; the second comparison
checks that the hash implementation is unchanged. Together they establish
that the patch hash chains are unchanged. Remove the temporary worktrees and
comparison files:

```sh
git --no-pager worktree remove "$QA_ROOT/order"
git --no-pager worktree remove "$QA_ROOT/layout"
rm "$QA_ROOT/patches.go" "$QA_ROOT/order.json" "$QA_ROOT/layout.json"
rmdir "$QA_ROOT"
```

Check generation writes only to the relocated asset tree. With the normal Go
build prerequisites installed, run from the layout checkout:

```sh
go generate ./domain/schema/...
git --no-pager diff --exit-code -- domain/schema/dqlite
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

Prerequisite ordering PR: #23890. Its QA steps also describe a
before-and-after comparison of the controller and model database schemas.

~Issue: no issue reference.~

~Jira card: not applicable.~
