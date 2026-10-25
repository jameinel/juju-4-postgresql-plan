# PostgreSQL PoC: handoff, Stage 6 remaining failures (2026-10-25)

> Point-in-time note. It supersedes `2026-10-25-stage-6-test-sql.md`.
> Fold the items below into the stage files, then mark this note
> superseded.

Goal: finish Stage 6 (PostgreSQL test harness and suites) so that
Stage 8 can continue. Follow the PLAN.md session protocol:

- Ask before design choices.
- Classify each change as bug, latent or portability (UPSTREAM-FIXES.md).
- Make one commit per concern on the owning stage branch, then restack.
- Run PostgreSQL tests one package at a time.
- Record every new dialect difference in DIALECT.md as soon as it is
  found, and let the user review the entry before fixing.
- While a long test run is going, check on it at least every two
  minutes (no sleeps over about 120 seconds), and stop a run that has
  stalled.

## 1. Where things are

The code tree is clean on `pg-poc/06-test-harness`. The stack is linear,
`go build ./...` passes on 08, and all `pg-poc/*` branches are pushed.

| Branch | Tip |
|---|---|
| `pg-poc/04-schemagen` | `0d83b530b9` |
| `pg-poc/05-queries` | `d658b402a6` |
| `pg-poc/06-test-harness` | `eac393da16` |
| `pg-poc/07-config` | `38344a18af` |
| `pg-poc/08-dbaccessor` | `2e67edc999` |

Done since the previous handoff (commit tables in the stage files):

- Stage 5: X34, X33, X35 (keymanager `RETURNING`, D36, U13) and X36
  (objectstore single `ON CONFLICT`, D37).
- Stage 6:
  - X6 single quotes (49 files, U6);
  - X7 date functions and `INSERT OR IGNORE` / `OR REPLACE` in tests;
  - U12 relation status ids as `INT` (D30);
  - X30 crossmodelrelation test;
  - the `core/database/dialect` leaf package (D11 amended), which fixes
    the harness import cycle and makes the `core/database/schema` tests
    run on both backends.
- DIALECT.md: X35 to X38 added, statuses updated. DECISIONS.md: N9, N10,
  D24 to D37. UPSTREAM-FIXES.md: U6, U10 to U13 and a portability list.

Restacking note: when 05 gains commits and 07 / 08 hold copies of older
06 commits, rebase 06 onto 05 first, then
`git rebase --update-refs --onto pg-poc/06-test-harness <old-06-tip-in-07> pg-poc/08-dbaccessor`,
and check `git merge-base --is-ancestor` for each pair.

## 2. Stage 5 work open (production SQL)

Commit each separately on `pg-poc/05-queries` as portability, then
restack.

1. **X37** `==`: change to `=` in
   - `domain/keymanager/state/state.go` `deleteUnusedUserKeys`
     (`HAVING count(...) == 0`; 6 PostgreSQL failures in
     `domain/keymanager/state`);
   - `domain/storageprovisioning/state/volume.go` (about line 196,
     `LEFT JOIN machine m ON sva.net_node_uuid == m.net_node_uuid`);
   - `domain/crossmodelrelation/state/model/offer.go` (about line 107,
     `cr.scope_id == 1`).

   The generated PostgreSQL triggers already turn the `==` in
   `domain/schema/modeltriggers.go` into `=`.
2. **X38** comma join scope: `domain/removal/state/model/storage.go`
   `GetDetachInfoForStorageAttachment` (3 failures). The user was
   reviewing the DIALECT.md entry; confirm the fix before applying it.
   The proposal is `FROM storage_attachment sa CROSS JOIN fulfilment f
   JOIN ...`; the alternative is putting the CTE first.
3. **X21** `PRAGMA defer_foreign_keys` in
   `domain/modelimport/state/model/import.go` (2 `internal/migration`
   failures and the model import suites). This needs a design decision:
   `DEFERRABLE` foreign keys from pgschemagen plus `SET CONSTRAINTS ALL
   DEFERRED`, or a dialect hook.

## 3. Latest PostgreSQL results

Logs:
`~/.copilot/session-state/5465307a-a217-4f78-8ff1-cc3b87cf91d7/files/`
(`restack/` is the latest run, on the current 06 tip).

| Package | Fails | Remaining causes |
|---|---|---|
| `domain/objectstore/state` | 0 | |
| `domain/keymanager/state` | 6 | X37 |
| `internal/migration` | 3 | X21 PRAGMA (2); `abort_test.go:233` SQLite `BEGIN ... END` trigger (1) |
| `domain/provisioner/state/model` | 5 | X25 test binds (see section 4) |
| `domain/provisioner` | 0 | |
| `domain/modelmigration/state/controller` | 0 | |
| `domain/removal/state/model` | 50 (was 371, then 185) | 25 X27; 22 X23 test; 3 X38; the rest test-side (section 4) |
| `domain/status/state/model` | 48 | Not diagnosed since the X23 run (`x23/`) |
| `core/database/schema`, `core/database/dialect`, `internal/database`, `core/watcher/eventsource`, `domain/schema` | 0 | |

Not rerun since the X6, X7 and X8 test commits; rerun each and record
the counts: `agent/agentbootstrap`, `domain/access/state`,
`domain/modeldefaults/state`, `domain/modelmigration/state/model`,
`domain/model/state/controller`, `domain/model/state/model`,
`domain/upgrade/state`, `domain/operation/state`, `domain/cloud/state`,
`domain/sequence/state`, `domain/machine/state`,
`domain/application/state` (about 17 minutes),
`domain/modelconfig/state`, `domain/controllernode/state`,
`domain/changestream/state`, `domain/lease/state`,
`domain/macaroon/state`, `domain/modelimport`.

## 4. Stage 6 test commits still to make

Make these on `pg-poc/06-test-harness` as `latent (test)` commits, one
per dialect issue. None needs a design choice unless noted.

1. **X23**: `domain/removal/state/model/secret_test.go` binds integer
   revision ids to `TEXT` `secret_value_ref.revision_id` and
   `secret_deleted_value_ref.revision_id` (about lines 294-305; 22
   failures). Bind strings.
2. **X25**: bind `true` / `false`:
   - `domain/provisioner/state/model/state_test.go` `allocate_public_ip`
     (about line 375) and `read_only` (about 489, 512, 534, 562);
   - `domain/removal/state/model/state_test.go` `force` in the removal
     job inserts (about lines 88 and 129; 2 failures).
3. **X27**: truncate expected times to `time.Microsecond`: 25 removal
   schedule-removal tests, and `domain/macaroon/state`.
   **X28**: year 9999 instead of `time.Unix(math.MaxInt64, 0)` in
   `domain/macaroon/state`.
4. **X16** in test SQL: `application_test.go:277` `SELECT COUNT(*), uuid
   FROM unit WHERE life_id = 0` (`TestEnsureApplicationOnMultipleMachines`).
5. **X9**: `application_test.go:1413` `CONCAT('application_', ?)` cannot
   type its parameter; use `'application_' || ?` or bind the full name.
6. **`DATETIME` type in test DDL**: `application_test.go:1217` and
   `unit_test.go:1088` create `charm_external_reference` with
   `created_at DATETIME` ("type datetime does not exist"). Use
   `TIMESTAMP`, valid on both.
7. **`DROP TRIGGER` without `ON <table>`**: `model_test.go:781`
   (`TestIsControllerModelControllerModel`). PostgreSQL requires
   `DROP TRIGGER IF EXISTS name ON model`; check that SQLite accepts the
   chosen form, or branch on `s.SchemaDialect()`.
8. **Error text (X20 / X26)**: `machine_test.go:2423` matches SQLite's
   `FOREIGN KEY constraint failed`. Match both texts, or an error type.
   Also anchored `ErrorMatches` in `domain/model/state/model/state_test.go:229,255`.
9. **RAISE(ABORT) to `schematesting.FailingTriggerSQL`**: removal
   `relation_test.go:351` and `relationwithremoteofferer_test.go:123`
   (its `WHEN (SELECT ...)` needs the condition in the function body);
   machine `reprovision_test.go:1068`; application
   `application_refresh_test.go`. `internal/migration/abort_test.go:233`
   uses `AFTER UPDATE OF ... BEGIN ... END`: it may need a helper
   variant, so ask the user.
10. **X29**: `domain/access/state/permission_test.go` `ensurePermission`
    (`LEFT JOIN ... WHERE` with no `ON`).
11. **U5**: `domain/upgrade/state/state_test.go` node ids as text.
12. **X21**: machine `reprovision_test.go:354` `PRAGMA
    foreign_key_check`: run only when `s.SchemaDialect()` is SQLite, or
    `SkipOnPostgreSQL` (N10).
13. **modelimport** `typeof` checks (`domain/modelimport/import_test.go`
    about 188-285) are SQLite-only; guard by dialect.
14. Diagnose the 48 `domain/status/state/model` failures.

Then rerun each package on SQLite and PostgreSQL. Prove the PostgreSQL
path is used: with the URL
`postgres://nobody@127.0.0.1:1/none?connect_timeout=2` the suite must
fail to connect.

## 5. Deferred

- U11 / D35: crossmodelrelation secrets queries (X16, X30). The portable
  design is in `2026-10-25-stage-6-test-sql.md` section 6.

## 6. Documentation still owed

- Stage files 03, 04 and 08: new commits; 08 bootstrap passes on
  PostgreSQL.
- DIALECT.md: update X37 and X38 when fixed, and the X25, X26, X29 test
  sites.

## 7. Tooling

Unchanged from the previous handoff, section 8. The scripts are in
`~/.copilot/session-state/e1b8ccab-eefe-4bb9-a26f-d0c9097f66da/files/`
(`runpg.sh <logdir> <pkgs...>`, `failsum.sh <log>`). The PostgreSQL URL
is in `~/.config/juju-pg-poc/db-url`. Never run two PostgreSQL test
packages at once. `domain/removal/state/model` takes about 16 minutes on
PostgreSQL; follow a run's progress with the `--- PASS` / `--- FAIL`
counts and the log's modification time.

Push:

```
git push --atomic --force-with-lease origin \
  $(git for-each-ref --format='%(refname:short)' 'refs/heads/pg-poc/*')
```

Run `gci write --section standard --section default --section
"Prefix(github.com/juju/juju)"` on every edited Go file.
