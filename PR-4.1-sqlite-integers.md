# fix(schema): use integer types for lookup ids

Two schema fixes found while prototyping PostgreSQL compatibility, scoped to
upstream 4.1 without PostgreSQL dependencies.

1. Thirteen integer lookup-ID columns were declared TEXT (SQLite type affinity
   masked this), including `relation_status_type.id` and
   `relation_status.relation_status_type_id`. They are now INT, the generated
   4.1.0 export types use `int64`, and the 4.0.12 -> 4.1.0 model-import
   transform parses the legacy string IDs. Malformed or overflowing
   relation-status IDs halt the transform rather than silently becoming zero.
   NULL messages and timestamps are preserved.
2. `v_revision_updater_application_unit` grouped by `u.uuid` instead of
   `a.uuid`, so it returned a row per unit and merged all unit-less
   applications into one NULL group.

The relation-status fix is a separate commit. Export, import and recovery
outputs were regenerated with `make rebuild-export`; only the expected export
types and transform changed.

The base schema files are edited in place for the unreleased 4.1 schema.
In-place controller upgrades are supported only within the same major.minor
version: a 4.0 controller cannot be upgraded to 4.1. Instead, bootstrap a
separate 4.1 controller, which creates a new controller database with the
corrected controller-column types, and migrate the 4.0 models to it. The 4.0
controller database is neither reused nor upgraded. On model import, the
exported data is transformed and inserted into a new model database using
the corrected 4.1 schema; the source model database is not upgraded in place.
These changes do not provide an in-place conversion for existing 4.1 beta
databases, and do not relax schema-patch hash requirements for supported
patch-version upgrades.

Not backported to 4.0: SQLite masked the lookup-ID type mismatches, and
migrated models get the corrected schema on import. The revision-updater fix
is also scoped to 4.1.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
  (Go tests exercise real dqlite import/export and recovery; live
  bootstrap/migration QA below has not been run)
- [x] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages

## QA steps

Automated checks completed:

```sh
go test -p 2 -race ./domain/schema/... ./domain/modelimport/... ./domain/export/... ./domain/recovery/... ./domain/status/state/model ./domain/relation/state ./domain/crossmodelrelation/state/model -count=1
make rebuild-export
make pre-check
```

The final import and recovery changes were rerun with `-race`. Race-enabled
stress runs completed 20 iterations each of the legacy import,
current-version round-trip and model recovery suites with no failures.

`make pre-check` skips static analysis by default on this branch.
Repository-configured lint was run separately on the changed Go package
scopes:

```sh
golangci-lint run -j 2 --timeout 5m --new-from-rev=7c96d020d5 ./domain/modelimport/... ./domain/recovery/state ./domain/export/types/v4_1_0/...
```

Regression coverage includes all six relation-status IDs,
malformed/empty/fractional/overflowing legacy IDs, integer storage (not just
scan coercion), seeded lookup preservation, nullable metadata, YAML
boundaries, current-version import/export and export/decode/recovery.

Manual QA (not run in this update):

1. Fresh 4.1 deployment.

```sh
$ juju bootstrap lxd dst41
$ juju add-model fresh
$ juju deploy juju-qa-test   # declares config options; creates a machine
$ juju wait-for application juju-qa-test
```

   Once the application is deployed, run the "Type checks" below against the
   `fresh` model. For relation-status checks, also deploy two compatible
   charms and relate them so that relation-status rows exist.

2. Migrate a 4.0 model with string-typed IDs.

```sh
$ juju bootstrap lxd src40   # 4.0.16 client and controller
$ juju add-model moveme
$ juju deploy juju-qa-test
$ juju wait-for application juju-qa-test
$ juju migrate src40:moveme dst41
```

   To check legacy relation statuses, create an integration between compatible
   charms on the source before migrating. Verify `juju status` shows the
   units active, and that `juju debug-log -m dst41:controller` and
   `-m dst41:moveme` show no transform or import errors. Then run the
   "Type checks" below against the migrated `moveme` model: the stored values
   should be `integer` even though the 4.0 source stored the same IDs as text.
   Finally `juju add-unit juju-qa-test` to confirm the model is functional.

### Type checks

Add an SSH key to the controller model (required in 4.1 before `juju ssh`),
then open the REPL on the controller. `$` is the local shell and `$$` is the
shell on the controller machine:

```sh
$ juju add-ssh-key -m controller "$(cat ~/.ssh/id_ed25519.pub)"
$ juju ssh -m controller 0
$$ juju_db_repl
repl (controller)> .switch model-moveme
```

Use the name of the model under test in `.switch`. Run the queries after the
applications and integration exist: `typeof` only reports on existing rows.

Declared types:

```sql
SELECT name, type FROM pragma_table_info('machine_platform');
SELECT name, type FROM pragma_table_info('application_platform');
SELECT name, type FROM pragma_table_info('charm_config');
SELECT name, type FROM pragma_table_info('unit');
SELECT name, type FROM pragma_table_info('relation_status');
SELECT name, type FROM pragma_table_info('relation_status_type');
```

`os_id`, `type_id`, `password_hash_algorithm_id`, `relation_status_type_id`
and `relation_status_type.id` should be `INT`.

Stored values (NULL passwords report `null`, so they are filtered out):

```sql
SELECT typeof(os_id) FROM machine_platform;
SELECT typeof(os_id) FROM application_platform;
SELECT typeof(type_id) FROM charm_config WHERE type_id IS NOT NULL;
SELECT typeof(password_hash_algorithm_id) FROM unit WHERE password_hash_algorithm_id IS NOT NULL;
SELECT typeof(password_hash_algorithm_id) FROM machine WHERE password_hash_algorithm_id IS NOT NULL;
SELECT typeof(password_hash_algorithm_id) FROM application_agent WHERE password_hash_algorithm_id IS NOT NULL;
SELECT typeof(relation_status_type_id) FROM relation_status;
SELECT typeof(id) FROM relation_status_type;
```

Every row should report `integer`. Check that the migrated integration still
reports its expected status.

### Revision updater view

Create a model with one application that has two units and two applications
that have none:

```sh
$ juju add-model units
$ juju deploy juju-qa-test two           # allocates machine 0
$ juju wait-for application two
$ juju deploy juju-qa-test none-a --to 0
$ juju deploy juju-qa-test none-b --to 0
$ juju add-unit two --to 0
$ juju wait-for application two --query='len(units)==2'
$ juju remove-unit none-a/0 none-b/0
$ juju wait-for application none-a --query='len(units)==0'
$ juju wait-for application none-b --query='len(units)==0'
```

Placing every unit on machine 0 with `--to` avoids waiting on further machine
allocations.

Then, in `juju_db_repl` against that model (`.switch model-units`):

```sql
SELECT a.name, v.num_units FROM v_revision_updater_application_unit AS v JOIN application AS a ON a.uuid = v.uuid ORDER BY a.name;
```

Expect exactly three rows: `none-a` 0, `none-b` 0 and `two` 2. Before the fix
this returned a row per unit (`two` twice, each with a count of 1) and a single
merged row for the applications without units.

## Documentation changes

No user-facing CLI or API workflow changes. The developer-facing schema
package documentation explains the schema-evolution contract: cross-minor
moves require a newly bootstrapped controller, model migration imports
transformed data into the destination schema rather than upgrading source
databases in place, and supported patch upgrades must preserve applied schema
files and their hash chain. User-facing upgrade documentation is unchanged.
Regression coverage now compares every field copied by the lookup-ID deltas,
including nullable fields, checks NotValid classification of malformed IDs
through transform wrapping, and explicitly compares revision-updater view
row counts with application counts. The QA instructions include the additional
relation-status columns.

## Links

**Issue:** N/A

**Jira card:** N/A

