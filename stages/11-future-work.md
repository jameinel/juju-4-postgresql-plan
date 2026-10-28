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
- READ COMMITTED everywhere experiment: run every Juju transaction at
  READ COMMITTED instead of SERIALIZABLE (D7), and measure what breaks
  and what it gains.
  - Why: SSI predicate (SIRead) locks share one server-wide table
    (`max_pred_locks_per_transaction` × MaxBackends; 8,768 targets on
    the PoC server). In 08-delete-locks, two SERIALIZABLE readers that
    each read every table of one model schema exhausted it (53200) with
    no schema changes running. A long SERIALIZABLE transaction also keeps
    the SIRead locks of every SERIALIZABLE transaction that commits while
    it runs, across all schemas. READ COMMITTED takes no predicate locks
    and needs no 40001 retries. The D47 delete and schema `Ensure`
    already run at READ COMMITTED (D53, DIALECT.md X43).
  - Trial: the default isolation is one connection parameter
    (`default_transaction_isolation` in `internal/database/postgresql/driver.go`).
  - Risk: dqlite has one writer, so domain code may rely on read-then-write
    sequences being atomic: check-then-insert, uniqueness checks not
    backed by a constraint, counters and lost updates. At READ COMMITTED
    these become races. Existing tests are mostly single-threaded, so a
    green run proves little.
  - Scope it with: an audit of domain state methods for read-then-write
    paths and their fixes (`SELECT ... FOR UPDATE`, constraints,
    `ON CONFLICT`); concurrent stress tests of contended operations; a
    check that change_log ordering (D7's per-namespace advisory lock)
    still holds; and Stage 9 benchmark comparisons of throughput, retry
    rate and peak SIRead/heavyweight locks. Compare with the mixed option
    (READ COMMITTED or `READ ONLY DEFERRABLE` for pure reads only).
- Actual sharding across databases/clusters. D3's placement indirection
  already belongs to the PoC; only using it for sharding is deferred.
- Continuous dual-backend CI and PostgreSQL tracing. Current dual-backend
  acceptance and generated-schema freshness tests are still required.
- Upstream sqlair dialect support replacing driver workarounds (D5).

U11 cross-model secrets is not deferred: its dedicated task gates the
functional milestone. Upstream 4.1 fix extraction is ongoing, not Stage 11.
