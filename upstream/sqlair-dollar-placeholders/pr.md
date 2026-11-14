# fix: emit numbered $N placeholders to support PostgreSQL

SQLair writes each input expression as a named `@sqlair_N` placeholder and
passes the values as `sql.NamedArg`. PostgreSQL drivers such as pgx only
accept numbered `$N` placeholders, and PostgreSQL parses `@sqlair_0` as the
`@` operator applied to a column, so every SQLair query with inputs fails
on PostgreSQL (#189).

SQLair now writes numbered placeholders (`$1`, `$2`, ...) and passes the
values positionally. Inputs are numbered in order of first appearance in
the generated SQL. PostgreSQL binds `$N` by its number, whereas SQLite
treats `$N` as a named parameter and indexes it by first appearance; the
two agree only when the numbers follow appearance order. A value reused in
every row of a bulk insert keeps its first placeholder, for example
`VALUES ($1, $2), ($3, $2)`.

Do not write raw numbered placeholders such as `WHERE id = $1`: they can
silently bind to SQLair's inputs. `Prepare` normally rejects them, while
ignoring placeholders in string literals, double-quoted identifiers and
comments. An unquoted identifier such as `col$1` is also rejected; quote it
as `"col$1"`. The check rejects dollar-quote delimiters, nested block
comments and a backslash immediately before a single quote in a string
literal, because their interpretation can differ between databases.
Dollar-quote-like text in unquoted identifiers can also be rejected.

SQLite's bracket- and backtick-quoted identifiers are not parsed; quotes or
comment markers in them can hide raw placeholders from the check. Use
double-quoted identifiers instead.

A new `tests/postgresql` Go module runs SQLair against PostgreSQL through
`github.com/jackc/pgx/v5/stdlib`. It is separate so that SQLair does not
depend on pgx and the root module stays on Go 1.24; the test module requires
Go 1.26. This is the oldest version that is considered actively maintained,
both in Ubuntu LTS and upstream. pgx itself only claims to require Go
1.25. Its tests skip unless `SQLAIR_POSTGRESQL_TEST_URL` is set, and each test
runs in its own schema.  CI runs the module against a `postgres:16` service
with both the module's minimum Go version and the latest Go release, and
govulncheck scans it. The changed workflows use current action releases pinned
by commit SHA.  CONTRIBUTING.md describes how to run the tests locally.

Compatibility:

- The generated SQL and argument form change for every driver. The public
  API does not change.
- Drivers that do not accept `$N` are no longer supported. This includes
  SQL Server's go-mssqldb, which accepts `@name` and `@pN`. MySQL drivers
  did not work before and still do not.
- Queries with raw `$N` placeholders normally fail at `Prepare` instead of
  when the query runs, subject to the identifier-quoting limitation above.

An alternative keeps `@sqlair_N` as the default and adds `NewDBWithOptions`
with a per-database placeholder style:
[`placeholder-style-option`](https://github.com/canonical/sqlair/compare/main...jameinel:sqlair:placeholder-style-option).
It shares the first three commits of this branch. Its
[changes from the shared base](https://github.com/jameinel/sqlair/compare/numbered-placeholders...placeholder-style-option)
include the option and follow-up documentation and test changes; this branch
also has the same PostgreSQL test follow-up.

Fixes #189.

## Functional QA

Use a PostgreSQL server where the user can create schemas. We need
`SQLAIR_POSTGRESQL_TEST_URL` set to a valid database URL.
You can set one up with Juju, create a dedicated model:

```bash
export JUJU_CONTROLLER='<controller name>'
export SQLAIR_QA_MODEL="$JUJU_CONTROLLER:sqlair-qa"
juju add-model -c "$JUJU_CONTROLLER" sqlair-qa
juju deploy postgresql --channel 16/stable
juju deploy data-integrator --channel latest/stable --config database-name=sqlairqa
juju integrate postgresql:database data-integrator:postgresql
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
reused value, a query with a slice input followed by another input, and a
`Prepare` call with a raw placeholder. The same program is built once in
the root module with the SQLite driver and once in the `tests/postgresql`
module with the pgx driver.

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
	db := sqlair.NewDB(sqldb)

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

	_, err = sqlair.Prepare("SELECT &Item.* FROM item WHERE id = $1", Item{})
	fmt.Println(err)
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

Run it against SQLite and PostgreSQL:

```bash
go run ./qa sqlite3 :memory:
(cd tests/postgresql && go run ./qa pgx "$SQLAIR_POSTGRESQL_TEST_URL")
```

Both commands print:

```text
[{1 shared} {3 shared}]
cannot parse expression: column 37: raw placeholder "$1" not supported, use SQLair input expressions
```

Before this change, the PostgreSQL command fails with
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
