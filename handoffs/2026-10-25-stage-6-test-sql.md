# PostgreSQL PoC: handoff, Stage 6 test SQL (2026-10-25)

> Point-in-time note. It supersedes
> `2026-10-24-stage-6-cleanup.md`. Fold the items below into the stage
> files and DECISIONS.md, then mark this note superseded.

Goal: finish Stage 6 (PostgreSQL test harness and suites) so that
Stage 8 can continue. Follow the PLAN.md session protocol:

- Ask before design choices.
- Classify each change as bug, latent or portability (UPSTREAM-FIXES.md).
- Make one commit per concern on the owning stage branch, then restack.
- Run PostgreSQL tests one package at a time.
- Record every new dialect difference in DIALECT.md as soon as it is
  found.

## 1. Where things are

The code tree is clean on `pg-poc/06-test-harness`. The stack is linear
and builds (`go build ./...` on 08), and the generated PostgreSQL DDL is
current. Push state is in section 8.

| Branch | Tip |
|---|---|
| `pg-poc/03-driver` | `7026e7d58e` |
| `pg-poc/04-schemagen` | `0d83b530b9` |
| `pg-poc/05-queries` | `a8301140de` |
| `pg-poc/06-test-harness` | `05b9567508` |
| `pg-poc/07-config` | `afc67094fa` |
| `pg-poc/08-dbaccessor` | `04939bfec8` |

New since the previous handoff:

- Stage 4:
  - `fix(database): scan postgresql timestamptz values in utc` (X14)
  - `fix(schema): use boolean for postgresql allocate_public_ip` (X25)
- Stage 5:
  - X16 change stream, status `Null[bool]`, X24 CTE in-lists, X18
    savepoints.
  - `fix(model): replace iif in the model connection info query` (X7)
  - `fix(blockcommand): order blocks by type instead of rowid` (X31).
    It adds `TestGetBlocksOrderedByType`, which fails with `rowid`.
- Stage 6:
  - U8/X11 BLOB columns, U7 `v_model_config`, U9 `removal.force`
    BOOLEAN.
  - `TestTxnLogging` skipped on PostgreSQL.
  - `test(schema): seed controller api port as text` (X23).
  - `test(domain): quote the user table in test sql` (X8, 14 files).

Note: 07 and 08 had been left on an older 06 tip. They were rebased onto
the current 06 with `--onto`. A plain `rebase --update-refs` from 08
only moves refs that point into the rebased range, so check
`git merge-base --is-ancestor` for each pair after restacking.

## 2. Confirmed decisions not yet in DECISIONS.md

- **N9**, the test-schema convention:
  - `juju-test-<pid>-<n>[-<suffix>]` names.
  - A `COMMENT ON SCHEMA` with the PID, start time and test name.
  - A once-per-process sweep of schemas whose PID is gone.
  - Quoting with `pgx.Identifier`.
- **N10**: backend-only skips use `SkipOnPostgreSQL` with a "dqlite only:
  <reason>" message, called before `SetUpTest`.
- **D24 onwards**:
  - X16 change stream: a `MAX(id)` subquery joined back.
  - `TestTxnLogging` is skipped on PostgreSQL rather than emitting tracer
    spans from the pgx wrapper.
  - X14: the driver decodes `timestamptz` in UTC.
  - X11: BLOB for random key bytes, extended to the bakery and root key
    columns.
  - U7: `v_model_config` prefers `agent_version` values.
  - `allocate_public_ip`: a PostgreSQL BOOLEAN override, read as
    `sql.Null[bool]`.
  - `removal.force` is BOOLEAN in dqlite too, with an import delta.
  - X31: blockcommand orders by `block_command_type_id`. dqlite order
    changes from insertion order to type order.
  - X7: production `IIF` becomes `CASE`.
  - X33: `octet_length(CAST(x AS TEXT))`. `pg_column_size` or
    page-based sizing is an upstream follow-up, because state code has no
    backend awareness.
  - X34: compare bool parameters with `FALSE` / `TRUE`.
  - Crossmodelrelation secrets (X30 plus X16) are deferred (section 6).

## 3. Stage 5 work decided, not yet done

Commit each separately on `pg-poc/05-queries` as portability. Then
restack (section 8).

1. **X34**: `domain/operation/state/query.go` `operationsQuery`. Change
   `$queryFlags.has_* = 0` to `= FALSE` and `= 1` to `= TRUE`. Fix the
   `queryFlags` comment in `types.go` ("as ints for SQL").
2. **X33**: `domain/operation/state/prune.go`:
   - Change `computeTableSize` (`octet_length(%q)`) to
     `octet_length(CAST(%q AS TEXT))`.
   - Apply the same to the literal `octet_length(...)` terms in
     `computeObjectStoreSize`. `osm.size` is an integer, and the other
     terms are safe either way.
   - Tests round to KiB, so their expectations should hold. Verify on
     both backends.
3. Add both to UPSTREAM-FIXES.md (portability cleanups). Add the
   `pg_column_size` follow-up.

## 4. PostgreSQL results (run `s6pre`)

Logs are in
`~/.copilot/session-state/e1b8ccab-eefe-4bb9-a26f-d0c9097f66da/files/s6pre/`.
The run used the 06 tip before the X23 and X8 commits, and before the
Stage 5 IIF and rowid fixes. Rerun everything.

| Package | Fails | Causes | State |
|---|---|---|---|
| `agent/agentbootstrap` | 3 | X8 | Fixed by the X8 commit |
| `domain/access/state` | 53 | X6 (`"dummy"`, `""` from `domain/model/state/testing/model.go:113,122` and about `:240`; `"fluffy"`, `"test"`, `"prod"`, `"test-endpoint"`, `"foobar"` in the permission, user and import tests), X8, X29 (`permission_test.go` `ensurePermission`) | X8 fixed; X6 and X29 open |
| `domain/modeldefaults/state` | 28 | X6 via `model/state/testing/model.go` (U6) | Open |
| `domain/modelmigration/state/model` | 1 | X6 double-quoted UUID, `TestGetMachineInstanceIDs` | Open |
| `domain/model/state/controller` | 63 | X23 (56), X7 `datetime` and `iif` in tests (`redirect_test.go:53`, `dbdeletion_test.go:21`), production IIF | X23 and IIF fixed; X7 tests open |
| `domain/model/state/model` | 2 | X26 anchored `ErrorMatches` (`state_test.go:229,255`) | Open |
| `domain/upgrade/state` | 3 | X6 `"foo"`, U5 integer node ids (`state_test.go:154-318`) | Open |
| `domain/provisioner/state/model` | 7 | X25: binds `1` to `allocate_public_ip` (`state_test.go:375`) and to `read_only` (`:489,512,534,562`); `INSERT OR IGNORE` (`package_test.go:124`) | Open |
| `domain/removal/state/model` | 371 | X6 `"test"`, `"iaas"`, `"prod"` in `state_test.go:179` (369); X25 bool encodes (2); `INSERT OR IGNORE` (`storage_test.go:2029,2058,2089`); RAISE sites | Open |
| `domain/operation/state` | 91 | X34 (36); X33 (8); X7 `datetime('now')` in test helpers (most of the 50 with no matched error, so check); X6 UUIDs; X27 in `TestImportOperationsWithSingleUnitTask` | X33 and X34 in section 3; the rest open |
| `domain/cloud/state` | 18 | X8 | Fixed by the X8 commit |
| `domain/sequence/state` | 0 | | Passes |
| `domain/machine/state` | 4 | X6 `"arm64"` (`placement_test.go:439`); `PRAGMA foreign_key_check` (`reprovision_test.go:354`, X21; skip or use a portable check); RAISE (`reprovision_test.go:1068`) | Open |
| `domain/status/state/model` | 60 | X23 Go ints bound to `TEXT` (9); X6 `"0"`, `"test"`, `"10.0.0.0/24"`; 18 `StatusInfo` / `Application` value mismatches, not diagnosed (X27 or X14?) | Open |

Not run in this pass:

- `domain/application/state` (about 17 minutes): `ep_names` syntax
  error, not diagnosed.
- `domain/modelconfig/state` (U7, now fixed).
- `domain/controllernode` and `domain/changestream` (X16, now fixed).

## 5. Stage 6 test-SQL commits still to make

Make these on `pg-poc/06-test-harness` as `latent (test)` commits, one
per dialect issue. They are mechanical and need no design choice.

1. X6: single-quoted literals. Start with U6
   `domain/model/state/testing/model.go`, then the packages above.
2. X7: `datetime` / `iif` in test SQL (model controller, operation).
   Bind times or use `CURRENT_TIMESTAMP`.
3. `INSERT OR IGNORE` in tests becomes `ON CONFLICT DO NOTHING`
   (provisioner, removal).
4. X25: bind `true` / `false` for boolean columns (provisioner, removal).
5. X23: bind strings for `TEXT` columns (status).
6. X26: `.*` around the message in anchored `ErrorMatches` (model/state).
7. X29: `ensurePermission`. Use an inner join with a `WHERE`, or
   `CROSS JOIN`.
8. RAISE(ABORT) to `schematesting.FailingTriggerSQL`: removal
   `relation_test.go:351`, `relationwithremoteofferer_test.go:127`;
   machine `reprovision_test.go:1068`; application
   `application_refresh_test.go`.
9. U5: upgrade node ids as text.
10. Machine `PRAGMA foreign_key_check`: `SkipOnPostgreSQL` (N10) or a
    portable assertion.
11. Optional: X27 (truncate to microseconds) and X28 (year 9999
    sentinel) in macaroon and operation tests.

Then rerun each package on SQLite (`-race` where verified) and on
PostgreSQL. Prove the PostgreSQL path is used: with the URL
`postgres://nobody@127.0.0.1:1/none?connect_timeout=2` the suite must
fail to connect.

## 6. Deferred: crossmodelrelation secrets (U11, latent)

`domain/crossmodelrelation/state/model/secrets.go` (about lines 36-44
and 80-96). Both queries:

- derive the application name with `substr(unit_name, 1,
  instr(unit_name, '/') - 1)` (X30);
- use `GROUP BY sruc.secret_id HAVING sruc.current_revision <
  MAX(sr.revision)` with bare `sr.uuid` / `sruc.current_revision` (X16).

With several consuming units, SQLite picks an arbitrary unit row. This is
a latent bug, so changing it is beyond a dialect fix. Record it in
UPSTREAM-FIXES. A portable design for later:

- Match with `substr(sruc.unit_name, 1, length(app.name) + 1) =
  app.name || '/'`.
- Drop the GROUP BY and use `sr.revision > sruc.current_revision`.
- Query 2: `SELECT DISTINCT sruc.secret_id`, plus the optional
  `sr.uuid IN (...)` filter.
- Query 1: add `sr.revision = (SELECT MAX(revision) FROM
  secret_revision WHERE secret_id = sr.secret_id)`.
- Semantics become "any consuming unit is behind". This needs multi-unit
  tests. In the test harness, a synthetic application's name is its UUID
  and consumer units are `<uuid>/0`.

## 7. Documentation still owed

- **UPSTREAM-FIXES.md**:
  - U10: blockcommand `rowid` (portability, fixed on Stage 5).
  - U11: crossmodelrelation secrets (open, section 6).
  - A portability-cleanups section: X16 change stream, IIF, `v_model`
    `ON TRUE`, X33, X34.
  - U5 and U6 statuses.
- **DECISIONS.md**: section 2.
- **DIALECT.md**: keep it current as fixes land (X6, X25, X26, X29 test
  sites).
- **Stage files**:
  - 03 and 04: the new commits.
  - 05: commit table.
  - 06: commit table, findings, pool cap.
  - 08: bootstrap passes on PostgreSQL.
- **PLAN.md**: the status table and the "Next" line.

## 8. Tooling

The scripts are in
`~/.copilot/session-state/e1b8ccab-eefe-4bb9-a26f-d0c9097f66da/files/`:

- `runpg.sh <logdir> <pkgs...>` honours `PKG_TIMEOUT`, `GO_TIMEOUT` and
  `GO_FLAGS`, and appends to `<logdir>/summary.txt`.
- `failsum.sh <log>` prints the first error per failing test. "?" means
  nothing matched.
- The PostgreSQL URL is in `~/.config/juju-pg-poc/db-url`.
- Never run two PostgreSQL test packages at once.

Restack after Stage 5 commits:

```
git switch -q pg-poc/08-dbaccessor
git rebase -q --update-refs pg-poc/05-queries
for p in 05-queries:06-test-harness 06-test-harness:07-config \
    07-config:08-dbaccessor; do
  git merge-base --is-ancestor pg-poc/${p%%:*} pg-poc/${p##*:} || echo "BROKEN $p"
done
go build ./...
```

Push:

```
git push --atomic --force-with-lease origin \
  $(git for-each-ref --format='%(refname:short)' 'refs/heads/pg-poc/*')
```

Run `gci write --section standard --section default --section
"Prefix(github.com/juju/juju)"` on every edited Go file.
