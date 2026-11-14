# feat: add a per-database placeholder style option

SQLair writes each input expression as a named `@sqlair_N` placeholder and
passes the values as `sql.NamedArg`. PostgreSQL drivers such as pgx only
accept numbered `$N` placeholders, and PostgreSQL parses `@sqlair_0` as the
`@` operator applied to a column, so every SQLair query with inputs fails
on PostgreSQL (#189).

`NewDBWithOptions` creates a `DB` from a `sql.DB` and a `DBOptions` value.
Its `PlaceholderStyle` field selects how inputs are written in the
generated SQL:

- `AtNamedPlaceholders`: `@sqlair_0` with `sql.NamedArg` values. This is
  the zero value and what `NewDB` uses, so existing code is unchanged.
- `DollarPlaceholders`: `$1` with positional values, for PostgreSQL.
- `ColonPlaceholders`: `:sqlair_0` with `sql.NamedArg` values.

`NewDBWithOptions` returns an error for a nil `sql.DB` or an unknown style.
Queries on the `DB` and its transactions use its style.

Inputs are numbered in order of first appearance in the generated SQL.
PostgreSQL binds `$N` by its number, whereas SQLite treats `$N` as a named
parameter and indexes it by first appearance; the two agree only when the
numbers follow appearance order. A value reused in every row of a bulk
insert keeps its first placeholder, for example `VALUES ($1, $2), ($3, $2)`.

`Prepare` rejects queries that contain their own numbered placeholder, such
as `WHERE id = $1`, which would otherwise bind to SQLair's first input with
`DollarPlaceholders`. The check applies to every style, so a statement can
be used with any `DB`. Placeholders inside string literals, quoted
identifiers and comments are ignored. An unquoted identifier such as
`col$1` is also rejected; quote it as `"col$1"`. Dollar-quoted strings are
not parsed, so `$1` inside one is rejected.

There is no style for `?` placeholders. A `?` placeholder cannot be reused,
so SQLair would need to repeat values for each use, and the drivers that
need it, such as MySQL drivers, are not tested with SQLair. There was an
attempt at implementing support, but it overly complicated the design.

A new `tests/postgresql` Go module runs SQLair against PostgreSQL through
`github.com/jackc/pgx/v5/stdlib`. It is separate so that SQLair does not
depend on pgx and the root module stays on Go 1.24; the test module requires
Go 1.26. Its tests skip unless
`SQLAIR_POSTGRESQL_TEST_URL` is set, and each test runs in its own schema.
CI runs the module against a `postgres:16` service, and govulncheck scans
it. CONTRIBUTING.md describes how to run it locally.

Compared with always emitting `$N`
([`numbered-placeholders`](https://github.com/canonical/sqlair/compare/main...jameinel:sqlair:numbered-placeholders)),
this keeps the SQL sent to existing drivers unchanged, including SQL
Server's go-mssqldb, at the cost of a public option, a code path per style
and tests for each style. Both branches share the numbering, raw
placeholder and PostgreSQL test commits.

Fixes #189.

## Functional QA

Use a PostgreSQL server where the user can create schemas. For example, with
Juju, a controller that can deploy machine charms, and `jq`, create a
dedicated model:

```bash
export JUJU_CONTROLLER='<controller name>'
export SQLAIR_QA_MODEL="$JUJU_CONTROLLER:sqlair-qa"
juju add-model -c "$JUJU_CONTROLLER" sqlair-qa --no-switch
juju deploy -m "$SQLAIR_QA_MODEL" postgresql --channel 16/stable
juju deploy -m "$SQLAIR_QA_MODEL" data-integrator --channel latest/stable \
    --config database-name=sqlairqa
juju integrate -m "$SQLAIR_QA_MODEL" postgresql:database data-integrator:postgresql
juju wait-for application -m "$SQLAIR_QA_MODEL" postgresql \
    --query='status=="active"' --timeout=20m
juju wait-for application -m "$SQLAIR_QA_MODEL" data-integrator \
    --query='status=="active"' --timeout=20m
```

The following obtains the relation credentials without printing them and
builds a URL for the first endpoint. It assumes this disposable deployment
does not have TLS enabled and that the endpoint is reachable from the test
machine. Use the appropriate TLS settings for other deployments.

```bash
set -o pipefail
SQLAIR_POSTGRESQL_TEST_URL="$(
    juju run -m "$SQLAIR_QA_MODEL" data-integrator/leader get-credentials --format json |
        jq -er '[.[] | select(.status == "completed") | .results.postgresql |
            select(.username and .password and .endpoints and .database)] |
            if length == 1 then .[0] |
                "postgres://\(.username | @uri):\(.password | @uri)@\(.endpoints | split(",")[0])/\(.database | @uri)?sslmode=disable"
            else error("expected one PostgreSQL credential set") end'
)"
export SQLAIR_POSTGRESQL_TEST_URL
```

Alternatively, export a suitable existing PostgreSQL URL.

From the repository root, create a program that runs a bulk insert with a
reused value and a query with a slice input followed by another input,
using the style named on the command line. The same program is built once
in the root module with the SQLite driver and once in the
`tests/postgresql` module with the pgx driver.

```bash
mkdir qa tests/postgresql/qa
cat > qa/main.go <<'EOF'
package main

import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"os"

	"github.com/canonical/sqlair"
)

type Item struct {
	ID    int    `db:"id"`
	Label string `db:"label"`
}

type IDs []int

var styles = map[string]sqlair.PlaceholderStyle{
	"at":     sqlair.AtNamedPlaceholders,
	"dollar": sqlair.DollarPlaceholders,
	"colon":  sqlair.ColonPlaceholders,
}

func main() {
	sqldb, err := sql.Open(os.Args[1], os.Args[2])
	if err != nil {
		log.Fatal(err)
	}
	defer sqldb.Close()
	// A temporary table is only visible on the connection that created it.
	sqldb.SetMaxOpenConns(1)
	ctx := context.Background()
	if _, err := sqldb.ExecContext(ctx, "CREATE TEMP TABLE item (id integer, label text)"); err != nil {
		log.Fatal(err)
	}
	db, err := sqlair.NewDBWithOptions(sqldb, &sqlair.DBOptions{PlaceholderStyle: styles[os.Args[3]]})
	if err != nil {
		log.Fatal(err)
	}

	// The label is a single value reused in every row of the bulk insert.
	insert := sqlair.MustPrepare("INSERT INTO item (id, label) VALUES ($Item.id, $M.label)", Item{}, sqlair.M{})
	err = db.Query(ctx, insert, []Item{{ID: 1}, {ID: 2}, {ID: 3}}, sqlair.M{"label": "shared"}).Run()
	if err != nil {
		log.Fatal(err)
	}

	// The slice expands to several placeholders before the next input.
	query := sqlair.MustPrepare("SELECT &Item.* FROM item WHERE id IN ($IDs[:]) AND label = $M.label ORDER BY id", Item{}, IDs{}, sqlair.M{})
	var items []Item
	if err := db.Query(ctx, query, IDs{1, 3}, sqlair.M{"label": "shared"}).GetAll(&items); err != nil {
		log.Fatal(err)
	}
	fmt.Println(items)
}
EOF
cp qa/main.go tests/postgresql/qa/main.go
cat > qa/sqlite.go <<'EOF'
package main

import _ "github.com/mattn/go-sqlite3"
EOF
cat > tests/postgresql/qa/pgx.go <<'EOF'
package main

import _ "github.com/jackc/pgx/v5/stdlib"
EOF
```

Run every style against SQLite, and the dollar style against PostgreSQL:

```bash
for style in at dollar colon; do go run ./qa sqlite3 :memory: "$style"; done
(cd tests/postgresql && go run ./qa pgx "$SQLAIR_POSTGRESQL_TEST_URL" dollar)
```

Each run prints:

```text
[{1 shared} {3 shared}]
```

The default style still fails on PostgreSQL:

```bash
(cd tests/postgresql && go run ./qa pgx "$SQLAIR_POSTGRESQL_TEST_URL" at)
```

It exits with
`ERROR: column "sqlair_0" does not exist (SQLSTATE 42703)`.

Run the PostgreSQL test module, then check that it left no test schemas
behind:

```bash
(cd tests/postgresql && go test -count=1 -v ./...)
psql "$SQLAIR_POSTGRESQL_TEST_URL" --tuples-only --no-align \
    --command "SELECT count(*) FROM pg_namespace WHERE nspname LIKE 'sqlair\_test\_%'"
```

`TestOutputs` and `TestInputs` pass, and the count is `0`. Without
`SQLAIR_POSTGRESQL_TEST_URL`, both tests are skipped.

Clean up:

```bash
rm -r qa tests/postgresql/qa
juju destroy-model "$SQLAIR_QA_MODEL" --destroy-storage --no-prompt
```

Only destroy the model if it was created for this QA.
