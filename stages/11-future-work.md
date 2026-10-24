# Stage 11: future work (not planned)

- Inception: a dqlite controller deploys PostgreSQL, then migrates via
  model export/import or a schema-aware copy.
- A controller charm relation to `postgresql` via `postgresql_client`, with
  credentials from a secret and rotation.
- Restoring a controller onto an existing PostgreSQL database (D23).
- Concurrency and change-stream alternatives (ANALYSIS.md section 10).
- Placement indirection for sharding models across databases or clusters.
- Dual-backend CI. Stage 4 already has a test that the generated PG schema
  is up to date.
- Upstream sqlair dialect support.
