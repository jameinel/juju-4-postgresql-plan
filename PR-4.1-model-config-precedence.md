Before this change, the `v_model_config` view combined `model_config` rows
with the `agent-stream` and `agent-version` values derived from the
`agent_version` table. If `model_config` also held either key, the view
returned two rows for that key, and `ModelConfig` (which builds a map from the
rows, last row wins) let row ordering choose the winner.

- The `agent-stream` branch used `UNION`, which sorts and de-duplicates. A
  competing `model_config` value that sorted after the real stream (e.g.
  `z-after` vs `proposed`) won.
- The `agent-version` branch used `UNION ALL`, so the winner was unspecified.

After this change, `model_config` rows for `agent-stream` and `agent-version`
are excluded from the view. `agent_version` is the only source, so each key has
exactly one value regardless of config contents or row order.

The 4.1 schema is still in beta (`4.1-beta3`) and no post-patches are
registered, so the base schema file `0027-agent-version.sql` is edited in place
rather than adding a `.PATCH.sql`.

Normal `SetModelConfig`, bootstrap and update validation already strip these
keys, but the v8 bulk importer bypasses those paths. The hand-written
`domain/modelimport/import.go` now removes both keys before the generated
bulk insert, without mutating the source payload. This covers transformed
4.0.12 and current-version payloads; the view guard remains defence in depth.

Migration bootstrap also used to read the stream from the payload's
`model_config`, defaulting to `released` when the key was absent. The
projection now resolves `agent_version.stream_id` through `agent_stream`
instead. A non-default stream is preserved even if config has no agent keys
or contains stale competing values. Missing stream references and empty
stream names are rejected before target-side writes rather than silently
choosing a default.

This closes an import boundary, not a demonstrated normal 4.0 write-path
leak. No schema-wide constraint, recovery replay change or beta-database
cleanup upgrade is included.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

### Unit tests

```sh
go test ./domain/modelconfig/... ./domain/schema/...
go test -race -p 2 ./domain/export ./domain/modelimport/... ./apiserver/facades/controller/migrationtarget ./internal/migration
```

`TestModelConfigAgentKeysAuthoritative` inserts competing `agent-stream` and
`agent-version` rows into `model_config`, with values that sort before and
after the authoritative ones. It asserts one effective value per key.

The import regressions verify ordinary config survives, agent keys are not
stored, the source payload is unchanged, and the effective config uses the
bootstrap-owned agent values. Projection tests cover absent/stale config,
non-default streams and malformed references. The v8 facade test exercises
4.0.12 decoding/transformation, and `TestModelImporterSuite/TestImportModel`
checks the real bootstrap/import flow stores `proposed` with no agent keys
in `model_config`, including when filtering removes every config row.

Follow-up validation from `a38cb5538d` to `c8b39bb2fd`: affected-package race
suites passed; configured Go lint reported zero issues; a 30-second bounded
migration stress run reported 228 runs and zero failures (timeout exit 124
was expected). `gofmt`, `gci` and `git diff --check` passed.

### Regression check (normal behaviour unchanged)

```sh
$ juju bootstrap lxd qa41
$ juju add-model m1
$ juju model-config -m m1 agent-stream agent-version
$ juju deploy juju-qa-test -m m1
$ juju model-config -m m1 agent-stream=proposed
$ juju model-config -m m1 agent-stream
```

Expect:

- `agent-stream` reports `proposed`.
- `agent-version` matches the controller version.
- No errors in `juju debug-log -m qa41:controller`.
- `juju model-config agent-version=...` is still rejected/ignored as before.

### Migration regression check (manual; not run)

With a supported 4.0 source controller `src40` and a target controller
`dst41` running this branch:

```sh
$ juju add-model -c src40 moveme
$ juju deploy -m src40:moveme juju-qa-test
$ juju model-config -m src40:moveme agent-stream=proposed
$ juju model-config -m src40:moveme agent-stream agent-version
$ juju migrate src40:moveme dst41
$ juju model-config -m dst41:moveme agent-stream agent-version
$ juju add-unit -m dst41:moveme juju-qa-test
```

Expect `proposed` and the recorded source target version after migration,
successful unit addition, and no migration errors in model/controller logs.
In the imported model database, `model_config` must have no `agent-stream`
or `agent-version` rows; `v_model_config` must expose the authoritative values.

### Exposing the precedence problem outside a unit test

The service layer prevents the competing rows from being created via the CLI.
To exercise the view guard independently of import filtering, write directly
to the model database on the controller machine:

```sh
$ juju ssh -m controller 0
# juju_db_repl            # dqlite REPL; select the model's database
> .switch <model-uuid>
> SELECT * FROM agent_version;
> INSERT INTO model_config (key, value) VALUES ('agent-stream', 'z-after'), ('agent-version', '99.0.0');
> SELECT * FROM v_model_config WHERE key LIKE 'agent-%';
```

Before (`4.1` without this PR):

- The view returns two rows per key.
- `agent-stream` typically yields both `proposed` and `z-after`, with `z-after`
  last because `UNION` output is sorted.
- `agent-version` yields both rows in no guaranteed order.
- `juju model-config -m <model> agent-stream agent-version` can report
  `z-after` / `99.0.0`.

After (this PR):

- The view returns exactly one row per key, taken from `agent_version`.
- `juju model-config` reports the `agent_version` values.

Clean up afterwards:

```sh
> DELETE FROM model_config WHERE key IN ('agent-stream', 'agent-version');
```

Caveats:

- This is a direct DB write that bypasses the service. A hand-edited model
  database is unsupported.
- The old behaviour depended on SQLite's row emission order, which is an
  implementation detail and not a contract. "Before" results are indicative
  rather than guaranteed, especially for `agent-version`.

## Documentation changes

No CLI syntax changes. Migration now preserves the authoritative agent stream
without relying on model config; malformed stream references are rejected.

## Links

**Jira card:** N/A
