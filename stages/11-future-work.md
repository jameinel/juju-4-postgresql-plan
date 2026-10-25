# Stage 11: deferred capabilities (not an execution stage)

Do not implement these opportunistically while making the PoC functional.
Record new evidence against the relevant item; a future task needs scope
and acceptance criteria first.

- PostgreSQL model import/export (D40): X21 FK deferral, modelimport
  backend-specific test SQL, and the `allocate_public_ip` export-type gap.
  Shared 4.1 fixes still require upstream import/export regression tests.
- Inception: a dqlite controller deploys PostgreSQL, then migrates via
  model export/import or a schema-aware copy.
- Controller-charm relation, secret-backed credentials and rotation.
  The current connection-source seam (N2) is required, not deferred.
- Restore onto existing PostgreSQL schemas (D23).
- PostgreSQL-primary failover and endpoint/reconnection experiments.
  Stage 10 tests Juju controller loss, not database failover. Read-replica
  routing is outside this PoC.
- Concurrency/stream alternatives and performance optimisation
  ([ANALYSIS.md section 10](../ANALYSIS.md#10-concurrency-alternatives-post-poc)).
  Benchmarking the current implementation is in scope in Stage 9.
- Actual sharding across databases/clusters. D3's placement indirection
  already belongs to the PoC; only using it for sharding is deferred.
- Continuous dual-backend CI and PostgreSQL tracing. Current dual-backend
  acceptance and generated-schema freshness tests are still required.
- Upstream sqlair dialect support replacing driver workarounds (D5).

U11 cross-model secrets is not deferred: its dedicated task gates the
functional milestone. Upstream 4.1 fix extraction is ongoing, not Stage 11.
