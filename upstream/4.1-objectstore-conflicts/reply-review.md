Addressed the sentinel suggestion: UUID reuse with different hashes or
size returns the documented `objectstoreerrors.ErrMetadataUUIDAlreadyExists`,
preserving the error text and remaining distinct from
`ErrHashAndSizeAlreadyExists`.

State tests use `ErrorIs` for different-hash and size-only conflicts, verify
unchanged metadata/paths/hints, and service tests cover wrapped propagation
through both put methods. The SQL-alias feedback is already fixed and its
thread resolved.

Validation: all objectstore Dqlite race suites pass; metadata stress reports
10 runs with zero failures before the intentional timeout; focused lint
reports zero issues. Deployment smoke and PostgreSQL tests were not run.
