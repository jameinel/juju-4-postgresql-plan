Thanks for the review.

- **SQL lint:** Removed the leading blank lines in all 11 reported files.
  SQLFluff 3.0.7 passes locally with the CI configuration.
- **FK validation connection:** Limited the in-memory validation database to
  one open connection, matching the first database used by the guard.
- **Filename ordering:** Documented lexical order in `readPatches`. Each base
  SQL file now has a distinct numeric position in the dependency order.
  Numbering in steps of ten leaves gaps for future prerequisites without
  renumbering subsequent files while gaps remain and schema evolution permits.
  The patch sequence and boundaries are unchanged; apart from the
  leading-newline fixes, their contents are unchanged.
- **Trigger diagnostics:** No change: the guard distinguishes later objects
  from missing ones when the authorizer reports them. SQLite can instead
  reject a missing object while preparing the trigger.

The QA recipe compares controller and model schema definitions; it does not
compare patch hashes or perform an upgrade pre-check. It documents the beta
hash-chain incompatibility separately. The live schema comparison was rerun
on the rebased head and again showed only the known comment whitespace
difference. Workload deployment and normal teardown were not established in
the earlier run.

I rebased onto 4.1 to resolve the merge conflict, preserving the upstream
agent-config view change in its reordered location.

The follow-up passed race-enabled SQLite and real-dqlite schema, schema-core
and agentbinary suites, scoped Go lint, `txncheck`, and export regeneration
without a generated diff.
