# fix: use numbered parameters for SQLair inputs

SQLair currently emits named `@sqlair_N` parameters. PostgreSQL's pgx
driver does not bind that syntax, so SQLair queries fail unless the SQL is
rewritten by a wrapper.

Emit `$N` placeholders and positional arguments instead. Assign numbers in
order of first appearance: PostgreSQL binds by number, whereas SQLite binds
these placeholders by appearance. Reuse a single placeholder when a bulk
insert repeats the same scalar value across rows.

This changes generated SQL and argument form for every driver without a
public API change. Drivers that do not accept `$N` can regress; SQL Server
users relying on `@name` may be affected, but that has not been verified.
MySQL and Oracle were not supported by the previous parameter syntax and
are not made compatible by this change. Real dqlite validation is pending.

Raw `$N` in SQLair queries is still passed through and can collide with
generated parameters. Do not mix it with SQLair inputs; rejecting it at
prepare time remains a follow-up before this is ready to merge.

The PostgreSQL test uses the isolated-schema fixture from the PostgreSQL
test-infrastructure dependency. This is one alternative to configurable
per-DB placeholder styles; a PR for the alternative has not yet been
published.

Addresses #189.

## Functional QA

From the branch checkout, configure `SQLAIR_POSTGRESQL_TEST_URL` with a
PostgreSQL URL whose role can create schemas. For example, if the URL is in
a private file:

```bash
export SQLAIR_POSTGRESQL_TEST_URL="$(<"$HOME/.config/juju-pg-poc/db-url")"
```

Run this example against both SQLite and PostgreSQL. It creates a unique
PostgreSQL schema, inserts two rows with one reused scalar parameter, and
drops that schema on exit.

```bash
qa_dir=$(mktemp -d ./sqlair-qa-XXXXXX)
cat > "$qa_dir/main.go" <<'EOF'
package main

import (
	"context"
	"database/sql"
	"errors"
	"fmt"
	"os"

	"github.com/canonical/sqlair"
	"github.com/canonical/sqlair/internal/pgtest"
	_ "github.com/mattn/go-sqlite3"
)

type Item struct {
	ID int `db:"id"`
}

func exercise(db *sql.DB) (result string, err error) {
	ctx := context.Background()
	if _, err := db.ExecContext(ctx, "CREATE TABLE items (id INTEGER, label TEXT)"); err != nil {
		return "", err
	}
	defer func() {
		_, dropErr := db.ExecContext(context.Background(), "DROP TABLE items")
		err = errors.Join(err, dropErr)
	}()
	insert := sqlair.MustPrepare(
		"INSERT INTO items (id, label) VALUES ($Item.id, $M.label)",
		Item{}, sqlair.M{},
	)
	if err := sqlair.NewDB(db).Query(ctx, insert,
		[]Item{{ID: 1}, {ID: 2}}, sqlair.M{"label": "shared"},
	).Run(); err != nil {
		return "", err
	}
	rows, err := db.QueryContext(ctx, "SELECT id, label FROM items ORDER BY id")
	if err != nil {
		return "", err
	}
	defer rows.Close()
	for rows.Next() {
		var id int
		var label string
		if err := rows.Scan(&id, &label); err != nil {
			return "", err
		}
		if result != "" {
			result += "; "
		}
		result += fmt.Sprintf("%d %s", id, label)
	}
	return result, rows.Err()
}

func run() (err error) {
	sqlite, err := sql.Open("sqlite3", ":memory:")
	if err != nil {
		return err
	}
	sqlite.SetMaxOpenConns(1)
	defer sqlite.Close()
	result, err := exercise(sqlite)
	if err != nil {
		return err
	}
	fmt.Println("sqlite:", result)

	ns, err := pgtest.Open(context.Background(), os.Getenv("SQLAIR_POSTGRESQL_TEST_URL"))
	if err != nil {
		return err
	}
	defer func() { err = errors.Join(err, ns.Close()) }()
	result, err = exercise(ns.DB)
	if err != nil {
		return err
	}
	fmt.Println("postgres:", result)
	return nil
}

func main() {
	if err := run(); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
EOF
go run "$qa_dir"
rm "$qa_dir/main.go"
rmdir "$qa_dir"
```

Expected output:

```text
sqlite: 1 shared; 2 shared
postgres: 1 shared; 2 shared
```

Check for leftover SQLair test schemas:

```bash
go run ./cmd/sqlair-pg-clean --older-than 1s
```

The example's schema should not appear. This command is a dry run; other
active or abandoned SQLair test schemas must not be removed without
identifying them. Its deletion mode uses `CASCADE`.
