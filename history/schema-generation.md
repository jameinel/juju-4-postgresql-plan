# Schema-generation exploration and original verification

Historical Stage 4 evidence, not current schema counts or a new task list.
Current contracts live in D10–D15/D30 and Stage 4. Later shared schema and
harness changes supersede parts of this snapshot.

## Initial implementation

The initial inventory contained four commits: schema bookkeeping,
generated/embedded PostgreSQL DDL, shared lookup/revision-count fixes
(U1/U2), and removal of obsolete PostgreSQL overrides. It produced 61
controller and 156 model patches, with zero-based filenames and one-based
bookkeeping versions.

The initial run reported fresh application and idempotent reapplication,
hash mismatch detection, rollback, FK column counts, seed data, view
queryability, guards, change events and blocked-writer ordering on
PostgreSQL 16.15. Seed timestamps were checked against the application
interval rather than literal equality between independently created DBs.

It also reported `go test -race` for schema, generator, import/export
packages, a six-run/two-process stress test and clean regeneration. Some
broader suites were later found to have lost the PG environment variable;
recover direct-backend evidence or rerun rather than assuming all historical
package claims establish PG coverage. Current shared-server guidance allows
one PostgreSQL test/stress process at a time.

## Object inventory at that point

Includes bookkeeping objects; excludes SQLite internal objects and
PostgreSQL constraint indexes.

| Namespace | Tables | Views | Explicit indexes | SQLite / PG triggers |
|---|---:|---:|---:|---:|
| controller | 95 | 20 | 90 | 89 / 90 |
| model | 251 | 59 | 193 | 291 / 292 |

The extra PostgreSQL trigger serialises change-log IDs. Gaps after
rollback are valid; IDs must not be assumed contiguous.

All 380 source triggers in that tree were Go-built. Earlier ANALYSIS.md
counts and statements about generated columns/REAL/GLOB/GROUP_CONCAT were
not a current inventory. Types then included 258 INT, 8 INTEGER,
38 DATETIME, 24 TIMESTAMP, 42 BOOLEAN and 2 BLOB columns. Later type fixes
change these numbers; do not hardcode them into acceptance tests.

## Shared corrections and override removal

Eleven lookup FK columns became integer-typed in shared DDL:
`cloud_credential.auth_type_id`,
`controller_node_password.password_hash_algorithm_id`,
`autocert_cache.encoding`, `model_life.life_id`,
`secret_permission.scope_type_id`, `charm_config.type_id`,
`machine.password_hash_algorithm_id`, `machine_platform.os_id`,
`application_platform.os_id`,
`application_agent.password_hash_algorithm_id`,
`unit.password_hash_algorithm_id`.

The change included export types, legacy numeric-string conversions,
nullable/invalid-input tests and revision-updater zero/multi-unit counts.
Removing obsolete override pairs/hunks left PostgreSQL output unchanged.
Other overrides handled predicate-free singleton joins, latest-login
aggregation, D17 ordering and duplicate constraint names.

The earlier TEXT relation-status workaround was superseded by D30/U12's
shared integer correction in Stage 6. Do not restore it based on this
history. Current beta base-schema edits require recreation of disposable
PoC databases; supported legacy payloads remain handled by import deltas.

## Dialect and change-log corrections

D11 initially put a dialect enum in `core/database/schema`. The test
harness needed it but could not import the engine without an import cycle.
The current leaf interface is in `core/database/dialect`, enforced to have
no Juju imports; the harness exposes `SchemaDialect()`.

D13 initially used identity defaults. PostgreSQL evaluates those before
BEFORE triggers, so the trigger could not distinguish supplied IDs and
consumed two sequence values per generated insert. The current plain
BIGINT/owned-sequence form allocates only for null IDs and advances past
explicit IDs, preserving changestream pruning fixtures.
