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
  all verification above passed. A second commit,
  `feat(scripts): limit disk use of the postgresql poc server`, sizes the
  disk and turns down logging and WAL retention (see findings).
- Restart notes:
  - Verified on 2026-10-23 against a Juju 3.6.29 LXD controller:
    postgresql 16/stable rev 1218 (PostgreSQL 16.15), data-integrator
    latest/stable, self-signed-certificates 1/stable. A fresh deployment
    settles in about 3 to 5 minutes. Re-running the script re-applies the
    postgresql configuration and rewrites the same URL.
  - On 2026-10-24 the first server filled its 5 GB root disk during Stage 6
    test runs. It was redeployed with the disk-use commit:

    ```sh
    scripts/postgresql-poc/deploy-postgresql.py -c "$PG_HOST_CONTROLLER" \
      -m pg-poc --root-disk-source juju-zfs -o ~/.config/juju-pg-poc/db-url
    ```

    The host controller is `lxd36-pgtest`; the server is container
    `juju-089db6-0` (10.158.110.70), login user `relation-4`. Only the
    plain model `pg-poc` exists. The TLS model `pg-poc-tls` and
    `db-url-tls` were not recreated; re-run with `-m pg-poc-tls -t <ca>`
    when TLS is needed.
  - URL and CA files live outside the repository in
    `~/.config/juju-pg-poc/`; re-run the script to regenerate them.
  - Watch disk use with `lxc exec <container> -- df -h /`. If it grows,
    check `/var/snap/charmed-postgresql/common/var/log/postgresql` (server
    logs) and `.../common/data/logs` (WAL) first.
  - Next: user review of the script. Since 2026-10-24 the branch sits in
    the line between 00 and 02, so every later stage carries the script.
- Findings (for later stages):
  - Profiles (2026-10-26): `--profile production|testing|disposable`
    (default `testing`). `disposable` sets `fsync=off` and
    `full_page_writes=off` via `patronictl edit-config`; the live `pg-poc`
    server runs it. Re-run with `-p testing` to restore durability.
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
  - Disk use (2026-10-24). The first server's 4.9 GB root filled with
    about 1.8 GB of server logs and 1.9 GB of WAL:
    - Server logs: pgaudit (`ROLE,DDL,MISC,MISC_SET`) and
      `log_statement = ddl` both logged the full DDL of every test schema.
      The logs also record each connection and disconnection. Files rotate
      every minute (`postgresql-%w_%H%M.log`) and are overwritten only a
      week later.
    - WAL: the charm sets `wal_keep_size` to 4096 MB. That keeps up to 4 GB
      for replicas, and this single unit has none. `archive_command` is
      `/bin/true`, so archiving holds nothing back, and there are no
      replication slots.
  - The script now deploys with `root-disk=20G`
    (`--root-disk`) and an optional `root-disk-source` (`--root-disk-source`,
    the LXD pool). It puts the `archive`, `data`, `logs` (WAL) and `temp`
    storage on `rootfs`, so one disk quota bounds the server logs, WAL and
    data. Without directives, charm storage goes to the model's
    `storage-default-filesystem-source`, which is `lxd`. On this host that
    is the LXD `default` pool, shared with other instances such as
    `apt-cache`. Juju's own `juju-zfs` and `juju-btrfs` LXD pools (30 GiB
    each, otherwise unused) back the `lxd-zfs` and `lxd-btrfs` storage
    pools.
  - Configuration set by the script, all from charm options:
    `plugin-audit-enable=false` (resets `pgaudit.log` to `none`),
    `logging-log-connections=false`, `logging-log-disconnections=false`
    and `durability-wal-keep-size=128`. With no replica nothing reads the
    kept WAL. 128 MB is Patroni's default. Patroni rejects values below
    16 MB (its validator is `IntValidator(min=16)`) and silently uses
    128 MB instead, so `0` has no effect.
  - Charm defect, rev 1218: `build_postgresql_parameters` reads the raw
    config, which holds both the new hyphenated option and the deprecated
    underscored one. Juju returns keys sorted and `-` sorts before `_`, so
    the deprecated option's default wins: `logging-log-connections=false`
    alone leaves `log_connections` on. Options read through the typed
    `CharmConfig` (pgaudit, the Patroni-managed `wal_keep_size`) prefer the
    hyphenated name. The script sets both spellings.
  - Not configurable through the charm, left as is by user choice:
    `log_statement = ddl` (hard-coded in the Patroni template, so DDL is
    still logged once) and `max_wal_size` (1 GB default). Both can only be
    changed with the `operator` superuser on the unit. Expect `pg_wal` to
    stay near `max_wal_size` plus `wal_keep_size`, about 1.1 GB.
