Addressed in follow-up commit `62a12cdc6a`:

1. The egress getter now orders CIDRs lexically in SQL, documents that
   contract at the state and service interface, and tests lexical order and
   relation isolation. The separate watcher initial query and fallback
   sources remain unordered; this change does not promise watcher-wide order.
2. Completed the `getRelationLife` comment.
3. Renamed the ingress test to `TestGetRelationNetworkIngressLexicalOrder`.

The affected state and service suites passed with `-race` on SQLite and real
dqlite. Scoped golangci-lint and txncheck passed. `make pre-check` ran but
skipped its full static analysis. Two-controller manual QA passed.
