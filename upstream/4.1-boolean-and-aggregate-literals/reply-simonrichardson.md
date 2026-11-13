Replaced the aggregate query with a single-row `LEFT JOIN` that selects the
instance ID directly. No matching row means the credentials did not match;
the nullable ID still distinguishes a matching but unprovisioned machine.
Both primary keys limit the join to at most one row.

The existing tests cover a valid match, wrong password or nonce, unknown
machine, and missing, NULL, or empty instance IDs. I also added checks that
wrong credentials on an unprovisioned machine return false without a
provisioning error. The affected SQLite and dqlite package suites pass.
