In follow-up commit `23034abf0e`, both state methods share the query and
row decoder. The model's stream check and binary query remain in the same
transaction; a generic helper is unnecessary because the row type is shared.

There is no `version` package shadowing: the imported package is
`semversion`, and the local `version` is the result of `semversion.Parse`.
Both methods return a contextual error for an invalid stored version rather
than substituting `version.Zero`; the existing tests cover that path.

The affected state and service suites passed with `-race` on SQLite; the
state suite also passed with `-race` on real dqlite. Scoped lint and
transaction analysis passed.
