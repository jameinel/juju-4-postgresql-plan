# Stage 07: config plumbing

Branch `pg-poc/07-config`, on `pg-poc/06-test-harness`.

- Goal: `db-backend` and `db-url` reach every controller's agent.conf.
- Steps:
  1. Controller config keys and validation (`controller/config.go`).
  2. instancecfg / agent.conf fields (`agent/agent.go`,
     `agent/format-2.0.go`).
  3. Bootstrap and newly provisioned controller propagation, including
     HA machines. No `agentconfigupdater` propagation: keys are immutable
     after bootstrap (D9).
  4. Connection-source interface (N2).
- Verification: unit tests round-trip all three keys through instancecfg
  and agent.conf; Stage 9 performs the same-tree default-dqlite bootstrap.
  Stage 10 validates live HA-node propagation. Earlier unit passes do not
  replace those runtime checks.
- Open questions: none. The PostgreSQL CA reaches every controller through
  a public `db-ca-cert` config key (PEM), D9; a later relation can replace
  it.
- Status: in review. Committed on `pg-poc/07-config`.
- Restart notes:
  - Done: controller keys `db-backend`, `db-url`, `db-ca-cert` (immutable,
    in `ControllerOnlyConfigAttributes` only, validated: `db-url` required
    iff postgresql); `agent.Config` `DBBackend/DBURL/DBCACert` and
    `SetDBConfig`, stored in agent.conf (`dbbackend`, `dburl`,
    `dbcacert`); set from instancecfg (new/HA machines) and in
    `cmd/jujuagentd/agent/bootstrap.go`; `internal/database/connsource`
    (`Source`, `Details`, `NewAgentConfigSource`). Agent mocks regenerated.
  - Not done, by design: `agentconfigupdater` does not propagate the keys
    because they are fixed at bootstrap (a change must not restart agents);
    CAAS `podcfg` / `caasapplication` do not set them (out of scope, D1).
    No `sslrootcert` file is written: Stage 8 applies `Details.CACert` in
    memory with `postgresql.WithRootCAs` (D22).
  - Not run: a live dqlite bootstrap; unit tests pass (agent, controller,
    instancecfg, connsource, jujuagentd/agent). `gci` was not installed at
    the time, so imports were ordered by hand; re-check with `gci` at
    review.
- Findings: Stage 8 should read connection details only through
  `connsource.Source`. `db-url` contains the password and is stored in
  controller config and agent.conf (N1).
