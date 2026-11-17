Agreed. The table and column names were fixed, but constructing SQL here
adds complexity without much benefit.

I replaced the dynamic sizing and counting helpers with static aggregate
queries. They measure variable-length text and use fixed per-row estimates
for UUIDs and non-text fields, while retaining the referenced object-store
content size. Nullable fields do not hide the rest of a row. The existing
rounding and average-based pruning decision are unchanged, although the
estimate can shift when pruning happens.

The fixed row allowances are constants and document the fields they cover. Tests
compare the estimate with representative content and flag changes to the
relevant table columns or types.

The operation-state suite passes on SQLite and real dqlite. The scoped
golangci-lint and transaction retry checker also pass.
