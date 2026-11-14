I've opened #<PR> to fix this. SQLair now writes numbered `$N`
placeholders, numbered in order of first appearance, and passes the values
positionally. PostgreSQL binds `$N` by number and SQLite binds it by first
appearance, so the same SQL works on both. It also rejects raw `$N`
placeholders at `Prepare`, and adds a separate `tests/postgresql` module,
run in CI, so that SQLair itself does not depend on pgx.

The main cost is compatibility. The SQL sent to every driver changes, and
drivers that do not accept `$N`, such as SQL Server's go-mssqldb, stop
working.

For comparison, the
[`placeholder-style-option`](https://github.com/canonical/sqlair/compare/main...jameinel:sqlair:placeholder-style-option)
branch keeps `@sqlair_N` as the default and adds
`NewDBWithOptions(sqldb, &DBOptions{PlaceholderStyle: DollarPlaceholders})`.
It shares the numbering, raw placeholder and test commits; its
[own change](https://github.com/jameinel/sqlair/compare/numbered-placeholders...placeholder-style-option)
is a public option, a code path per style and tests for each style. I have
not opened a PR for it.

Which direction would you prefer?
