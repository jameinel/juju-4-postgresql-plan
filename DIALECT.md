# SQLite and PostgreSQL differences

Behaviour that differs between dqlite (SQLite) and PostgreSQL and that
Juju code depends on. Each entry gives what each backend does, where Juju
hits it, and how the PoC handles it. Cite entries by id (X1, X2, ...).
Add an entry for every new difference found, including ones only
worked around. Keep the id when its handling changes.

Verified on SQLite 3.45.1 and PostgreSQL 16.15 on 2026-10-24 unless an
entry says otherwise. Schema-only differences handled by `pgschemagen`
are summarised in section 4 rather than listed one by one.

Handling states:

- **Shared fix**: one query or DDL valid on both backends.
- **Driver rewrite**: the Stage 3 driver (`internal/database/postgresql`)
  rewrites SQL that sqlair or callers produce. This is a PoC workaround;
  long term the query layer should emit dialect-correct SQL (D5:
  upstream sqlair dialect support), or callers should avoid the
  construct.
- **Generator**: `pgschemagen` translates it (Stage 4).
- **Open**: seen, not yet handled.

## 1. Query syntax

| Id | Construct | SQLite | PostgreSQL | Handling |
|---|---|---|---|---|
| X1 | Placeholders `?` and `@sqlair_N` | Native | Uses `$N` | Driver rewrite (Stage 3, D5). |
| X2 | Empty list `x IN ()` / `x NOT IN ()` | False / true, even for NULL `x` | Syntax error | Driver rewrite (Stage 3) to `x = ANY ('{}')` / `x <> ALL ('{}')`. |
| X3 | `ON CONFLICT DO UPDATE` with no conflict target | Allowed | "requires inference specification or constraint name" | Shared fix (Stage 5): name the target. |
| X4 | Unqualified column in `ON CONFLICT ... DO UPDATE` `SET` expression or `WHERE` | Means the target table | "column reference is ambiguous" (target or `excluded`) | Shared fix (Stage 5): qualify with the table name. |
| X5 | Two-argument scalar `max(a, b)` / `min(a, b)` | Scalar | Aggregate only; scalar is `GREATEST` / `LEAST`, which SQLite lacks | Shared fix (Stage 5): `CASE`. |
| X6 | Double-quoted string, `""` or `"abc"` | String literal when no column matches (legacy misfeature) | Identifier; `""` is "zero-length delimited identifier" | Shared fix: single quotes. Test SQL still has sites (U6, Stage 6 suites). |
| X7 | `IFNULL`, `GROUP_CONCAT`, `INSERT OR IGNORE`, `IIF`, `datetime('now', ...)` | Built in | Missing | Shared fix (Stage 5, D16): `COALESCE`, aggregate in Go, `ON CONFLICT DO NOTHING`, `CASE`, bind times or `CURRENT_TIMESTAMP`. Test SQL in `domain/model/state/controller` still uses `datetime` and `iif`. |
| X8 | `user` as an identifier | Allowed | Reserved, and silent in some positions: `FROM user` reads the `current_user` function as a one-row table, and `u.uuid` on that alias is the cast `uuid(u)` ("operator does not exist: uuid = text") | Shared fix (Stage 5): quote as `"user"`. Fourteen test files still use it bare (Stage 6 suites). |
| X9 | Parameter with no type context, such as `$1 IS NULL` | Allowed | "could not determine data type of parameter $N" | Open (Stage 6 suites): cast the parameter. |
| X24 | `x IN name` where `name` is a table or CTE, with no parentheses | Allowed, same as `x IN (SELECT * FROM name)` | Syntax error | Shared fix (Stage 5): `x IN (SELECT col FROM name)`. Eleven production sites in `domain/application/state` (`application_endpoint.go`, `k8sservice.go`), `domain/removal/state/model` (`unit.go`, `application.go`, `storage.go`), `domain/operation/state` (`task.go`, `prune.go`) and `domain/relation/state/relation.go`. Seen as `ep_names` errors in the Stage 6 suites. |

## 2. Types and values

| Id | Behaviour | SQLite | PostgreSQL | Handling |
|---|---|---|---|---|
| X10 | Type affinity | Stores any type in any column and compares across types without error | Assignment casts untyped literals; `text = integer` is "operator does not exist" | Shared fix per site: U5 (node ids as text), Stage 4 FK type corrections. More expected from the Stage 6 suites. |
| X11 | Invalid UTF-8 or NUL (`0x00`) in `TEXT` | Stored as given | "invalid byte sequence for encoding UTF8" (22021); NUL is rejected even though it is valid UTF-8 | Per column. `domain/schema/storage.go` `binaryTextColumns` lists the `TEXT` columns bound as bytes. Those holding raw random bytes failed on PostgreSQL, and all are fixed in one Stage 6 commit (U8): the dqlite columns become `BLOB`, which generates `bytea`. (1) `user_activation_key.activation_key` (32 bytes from `generateActivationKey`) broke `add-user` (5 `domain/access/state` failures). (2) The eight `bakery_config` key columns (32-byte keys via `keyScanner`) failed 3 `domain/macaroon/state` tests; `InitialiseBakeryConfig` runs in the bootstrap worker (`internal/worker/bootstrap/fresh.go`), so a PostgreSQL controller would have failed there. (3) `macaroon_root_key.root_key` (24 random bytes from `dbrootkeystore`) would have failed on the first login macaroon; the tests use ASCII keys. The other listed columns hold hex, base64, JSON or YAML, which are valid UTF-8. `domain/recovery/state` tests bind `{0, 128, 255}` to them (dqlite only, see X21). |
| X12 | `BOOLEAN` | Integer 0 / 1 | `boolean` | Generator (D14). `= false` / `= true` works on both. |
| X13 | `DATETIME` / `TIMESTAMP` | Text in whatever format was bound | `timestamptz`; zone-less text is read in the session `TimeZone` | Generator (D14); connections set `TimeZone=UTC` (Stage 5 findings, `WithTimeZoneUTC`). |
| X14 | Decoded `time.Time` location | UTC (go-dqlite parses in UTC; tests compare locations) | pgx returns `timestamptz` in `time.Local` | Driver (Stage 3): every connection registers a `timestamptz` codec with `ScanLocation: time.UTC`. Seen in `modelmigration` `TestSetLastLogTransferTimeMonotonic`. |
| X23 | Go value bound to a parameter of another type | Stored with affinity | pgx will not encode a Go integer for a `TEXT` parameter | Shared fix per site: bind the declared type (PLAN.md, testing notes). Open: `schematesting.SeedControllerTable` binds the integer `controller.DefaultAPIPort` to the `TEXT` `api_port` (56 failures in `domain/model/state/controller`). |
| X25 | Boolean values in `INT` columns | `TRUE` / `FALSE` are 1 / 0, and Go bools bind as integers | A boolean expression cannot be assigned to `bigint` (42804), and pgx will not encode a Go bool for `int8` or an integer for `boolean` | Per column. `constraint.allocate_public_ip` is `INT` in dqlite to avoid the go-dqlite nullable-bool bug and its table is `STRICT`, so a PostgreSQL override makes it `BOOLEAN` (Stage 4) and status reads it as `sql.Null[bool]` (Stage 5). `removal.force` is `NOT NULL` in a non-`STRICT` table, so the dqlite column becomes `BOOLEAN` (Stage 6, U9). Test SQL that binds `1` to `allocate_public_ip` (`domain/provisioner/state/model`) must bind `true`. |
| X27 | Timestamp precision | go-dqlite binds `time.Time` as ISO 8601 text with nanoseconds, so values round-trip exactly | `timestamptz` holds microseconds; pgx's binary encoding truncates the nanoseconds of both stored values and parameters | Open. Comparisons in SQL stay consistent because both sides are truncated, but a value read back is not `Equal` to the Go value written. Seen in `domain/macaroon/state` `TestFindLatestKeyEquality` and `TestInsertAndGetKey` (`time.Now()` keys). Tests should truncate to `time.Microsecond`. Production code that compares a read-back time with an in-memory one would differ too; none found yet. |
| X28 | Timestamps outside the `timestamptz` range | Any `time.Time` formats as text and compares as text | pgx's binary encoding computes microseconds since 2000 without overflow checks, so `time.Unix(math.MaxInt64, 0)` silently becomes a wrong timestamp; no error is raised | Open. Test only: `domain/macaroon/state` uses it as a "no upper bound" sentinel, so `TestFindLatestKeyCreatedAfter` and `TestFindLatestKeyExpiresAfter` find no key. Use a representable bound, such as year 9999. No production sentinel found. |

## 3. Unspecified behaviour SQLite makes look deterministic

| Id | Behaviour | SQLite | PostgreSQL | Handling |
|---|---|---|---|---|
| X15 | `UNION` row order | Sorted, as a side effect of de-duplication | Any order (hashing) | Open, fix agreed (U7, Stage 6): `v_model_config` returns two `agent-stream` rows when `model_config` also has the key, and `ModelConfig` keeps the last. Wrong on dqlite too when the config value sorts last. The view will exclude `agent-stream` and `agent-version` from its `model_config` branch and use `UNION ALL`. |
| X16 | Bare columns with `GROUP BY` | Allowed. With a single `MIN` / `MAX` aggregate, bare columns come from the row holding the min / max; otherwise from an arbitrary row | Error 42803 | Shared fix per site (D18). The change stream `selectQuery` (`internal/changestream/stream/stream.go`) relied on the `MAX` rule, so on PostgreSQL the stream worker died and watchers never went idle (5 `domain/controllernode` failures). It is a CTE of `MAX(id)` per namespace and changed value, joined back to `change_log` (Stage 5). `internal/worker/dbrepl/worker.go` has the same query (out of scope). |
| X17 | Order of rows without `ORDER BY` | Usually insertion or index order | Any order | Shared fix per site when found (D17 for constraint tags, spaces and zones). |

## 4. Schema and DDL (generator)

Handled by `pgschemagen` (Stage 4; D10, D12, D14, D15). Detail in
ANALYSIS.md section 4.

- Forward references: SQLite resolves `REFERENCES` and view sources on
  use; PostgreSQL at `CREATE`. Reordered by the generator (D10).
- A foreign key target must be a primary key, unique constraint or
  non-partial unique index when the key is created.
- `INTEGER PRIMARY KEY AUTOINCREMENT`, `STRICT`, `IIF`, `strftime`
  defaults.
- Triggers: SQLite `BEGIN ... END` bodies, `RAISE(ABORT | FAIL, ...)`
  and `WHEN` become PL/pgSQL functions with `RAISE EXCEPTION`.
  PostgreSQL does not allow subqueries in a trigger `WHEN`
  (`schematesting.FailingTriggerSQL`, Stage 6).
- PostgreSQL evaluates column defaults before `BEFORE` triggers (D13).
- Identifiers over 63 bytes are truncated by PostgreSQL; the generator
  adds a hash suffix (D14).

## 5. Transactions, errors and connections

| Id | Behaviour | SQLite / dqlite | PostgreSQL | Handling |
|---|---|---|---|---|
| X18 | Error inside a transaction | Only the failing statement is undone | The whole transaction is aborted | Shared fix per site: avoid the error in SQL. `PinLease` uses `ON CONFLICT DO NOTHING` (Stage 5). Stage 5 also fixes four writes that ignored a unique or primary-key error and then committed ("commit unexpectedly resulted in rollback"): `RequireMachineReboot`, `AllowCloudType` and the unit net node insert use `ON CONFLICT DO NOTHING`, and `EnsureExternalUser` looks the user up first. Under SERIALIZABLE, a key inserted concurrently after such a check is reported as 40001 and retried, not 23505. |
| X19 | Isolation | One writer at a time | SERIALIZABLE; 40001 / 40P01 at any statement or `COMMIT` | D7; retried by `drivererrors.IsErrRetryable` (Stage 3). |
| X20 | Constraint errors | SQLite extended codes | SQLSTATE 23xxx; unique and primary key share 23505 | Mapped in `drivererrors` (Stage 3). |
| X21 | Foreign keys | Off unless `PRAGMA foreign_keys`; `PRAGMA defer_foreign_keys` | Always on; deferral needs `DEFERRABLE` keys and `SET CONSTRAINTS` | Harness requires foreign keys on (Stage 6). Deferral open (Stage 5 findings). `domain/recovery/state` uses `PRAGMA defer_foreign_keys` and `PRAGMA foreign_key_check` to replay a dqlite dump. Recovery is dqlite only (PostgreSQL would use its own backups), so its 11 PostgreSQL failures are out of PoC scope. |
| X22 | Connections | In-process, cheap | Server `max_connections` (100 on the PoC server); an unbounded `database/sql` pool can exhaust it | `dbaccessor` `applyDBLimits` caps dqlite pools at 3 per database. The Stage 6 harness applies the same cap to PostgreSQL; the Stage 8 `pgWorker` must too. |
| X26 | Error text | The message alone, such as `model table is immutable, only insertions are allowed` | pgconn adds a prefix and suffix: `ERROR: <message> (SQLSTATE P0001)` | Anchored `ErrorMatches` patterns in tests need `.*` around the message (`domain/model/state/model` immutable-model tests, Stage 6). Production code matches errors by type, not text. |
