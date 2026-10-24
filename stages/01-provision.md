# Stage 01: PostgreSQL provisioning script

Branch `pg-poc/01-provision`, on `pg-poc/00-base-fixes`.

- Goal: a repeatable way to obtain a `db-url` for a PostgreSQL that LXD
  controllers can reach.
- Prerequisites: a Juju 3.6 controller on LXD.
- Steps:
  1. Add `scripts/postgresql-poc/deploy-postgresql.py` (N7) with
     `-c/--controller` and `-m/--model` (both required), an optional
     `-o/--output <file>` for the URL and an optional
     `-t/--tls-ca-file <file>` for TLS mode (D8).
  2. Deploy `postgresql --channel 16/stable` and
     `data-integrator --config database-name=juju`; integrate them; wait for
     active/idle. In TLS mode also deploy `self-signed-certificates` and
     integrate it with `postgresql:client-certificates`.
  3. Run the data-integrator `get-credentials` action and build
     `postgres://<user>:<password>@<host>:5432/juju?sslmode=...`
     (`disable`, or `verify-full&sslrootcert=<ca-file>` in TLS mode).
  4. Document teardown (`juju destroy-model`).
- Verification:
  - Connect with `psql` (or a small Go probe) from the host and from a fresh
    LXD container on the same bridge.
  - Confirm the user can `CREATE SCHEMA` and `DROP SCHEMA` in `juju`.
  - Note whether `pg_hba` admits the LXD subnet; if not, record the
    workaround.
  - Repeat the connection checks in TLS mode with `verify-full`.
- Open questions: none (resolved by D8, N5 and N7).
- Status: in review. The script is committed on `pg-poc/01-provision` and
  all verification above passed.
- Restart notes:
  - Verified on 2026-10-23 against a Juju 3.6.29 LXD controller:
    postgresql 16/stable rev 1218 (PostgreSQL 16.15), data-integrator
    latest/stable, self-signed-certificates 1/stable. A fresh deployment
    settles in about 5 minutes. Re-running the script is a no-op that
    rewrites the same URL.
  - Running deployments on `$PG_HOST_CONTROLLER`: model `pg-poc` (plain) and
    model `pg-poc-tls` (TLS). URL and CA files live outside the repository
    (the agent used `~/.config/juju-pg-poc/`); re-run the script to
    regenerate them.
  - Next: user review of the script. Since 2026-10-24 the branch sits in
    the line between 00 and 02, so every later stage carries the script.
- Findings (for later stages):
  - `pg_hba` has `host juju relation-N 0.0.0.0/0 scram-sha-256` (`hostssl`
    with TLS). The host (via the `lxdbr0` gateway) and LXD containers on
    the bridge connect; no workaround is needed.
  - The login user (`relation-N`; note the hyphen when quoting) is switched
    to `charmed_juju_owner` at login by the charm's `login_hook` extension.
    New schemas and tables are owned by `charmed_juju_owner`; the database
    is owned by `charmed_databases_owner`. `RESET ROLE` and `DISCARD ALL`
    revert to `relation-N` and lose the owner privileges, so the Stage 3
    driver and Stage 8 pools must not reset the session role, or must run
    `SET ROLE charmed_juju_owner` after connecting.
  - The `juju` database already has `public` and `login_hook` schemas;
    namespace schema names must not collide with them (Stage 8).
  - The server default isolation is `read committed`; D7 relies on the
    connection config (Stage 3, step 5).
  - Enabling TLS makes `pg_hba` `hostssl` only, so plain and TLS need
    separate models. Server certificates carry the unit IP as an IP SAN, so
    `verify-full` against the IP works. The self-signed CA is valid for one
    year. `sslmode=disable` and a wrong CA are both rejected.
  - Verification used `psql` with the URL split into `PG*` environment
    variables, keeping the password out of process arguments.
