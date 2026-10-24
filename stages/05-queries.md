# Stage 05: query portability fixes

Branch `pg-poc/05-queries`, on `pg-poc/04-schemagen`.

- Goal: make shared queries valid on both backends.
- Steps (locations in ANALYSIS.md section 5):
  1. `IFNULL` to `COALESCE`.
  2. `INSERT OR IGNORE` to `ON CONFLICT DO NOTHING`.
  3. Replace `GROUP_CONCAT` (aggregate in Go or dialect hook).
  4. Lease and change stream time arithmetic: compute in Go and bind.
  5. Quote `"user"` in queries.
- Verification: existing dqlite tests still pass; the same tests pass on PG
  once Stage 6 lands.
- Status: in review. One `fix(domain)` commit on `pg-poc/05-queries`.
  Affected domain, changestream and
  leaseexpiry suites pass on dqlite/SQLite; nothing was run on PG (needs
  Stages 4 and 6). `golangci-lint` not run.
- Restart notes:
  - Done: `IFNULL` -> `COALESCE` (5 sites); `INSERT OR IGNORE` -> `ON
    CONFLICT DO NOTHING`; `"user"` quoted in every query and test helper
    SQL found (credential, access, modelmigration, model, keymanager, ssh);
    `user` alias in `getUserWithAuth` renamed to `u`; `GetUnitsK8sPodInfo`
    aggregates ports in Go (see D16).
  - Time: lease `ClaimLease` / `ExtendLease` bind `start_at` / `expiry_at`
    (`YYYY-MM-DD HH:MM:SS` UTC, expiry rounded up to whole seconds) from a
    `clock.Clock` field on `lease/state.State` (wall clock, no constructor
    change); `leaseexpiry` binds the same format for its `expiry < ?`
    checks. Plain `DATETIME('now')` sites (removal storage status x2,
    upgrade `node_upgrade_completed_at`, change stream watermark x2) use
    `CURRENT_TIMESTAMP`, which SQLite renders identically, to avoid
    threading a clock through three state constructors.
  - Not done on purpose: `core/database/schema/query.go` (Stage 4);
    `PRAGMA defer_foreign_keys` (see findings).
- Findings (for later stages):
  - Bound lease timestamps are zone-less text. With D14 (`timestamptz`) PG
    interprets them in the session `TimeZone`, so the connection must run
    with `TimeZone=UTC` or leases expire at the wrong time. Stage 6 added
    `postgresql.WithTimeZoneUTC` for test connections; Stage 8 must use it
    for agent pools. `CURRENT_TIMESTAMP` is fine either way.
  - `PRAGMA defer_foreign_keys = 1` in `domain/recovery/state/load.go` and
    `domain/modelimport/state/model/import.go` has the PG equivalent `SET
    CONSTRAINTS ALL DEFERRED`, which only affects `DEFERRABLE` foreign keys.
    Stage 4 would have to emit FKs as `DEFERRABLE INITIALLY IMMEDIATE`
    (including D10's `ALTER TABLE ... ADD CONSTRAINT` ones). Not needed for
    bootstrap. Other SQLite-only code left for Stage 6/8: `pragma.*`
    (`foreign_keys`, `delete_database`), and `""` used as an empty string
    literal in `domain/model/state/testing/model.go` (`VALUES (?, ?, ?, "",
    true)`), which PG reads as an identifier.
  - Sites that look portable but are only checked on SQLite: SQLite lets
    `GROUP BY` leave columns bare and compares int to text freely; Stage 6
    will surface more.
  - Next: user review.
