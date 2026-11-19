SQLite accepts schema definitions that refer to tables or views created
later. This makes the source order misleading and prevents consumers that
require referenced objects to exist from applying the definitions in order.

This orders the controller and model schema definitions so foreign-key targets
and their keys exist before the referencing table, and view sources exist
before the view. Split out shared prerequisites where whole-file ordering
cannot satisfy the dependency graph. Preserve the existing definitions,
seed data and generated triggers.

This also adds a schema-order guard that checks table keys, view dependencies
and trigger dependencies, including references within a single patch. It uses
SQLite's schema metadata and resolver, not PostgreSQL-specific SQL or runtime
dependencies.

This changes the 4.1 beta base-schema hash chain. Earlier 4.1 beta
databases cannot upgrade in place across the change. Released 4.0 models
continue to reach 4.1 through migration; historical release DDL is unchanged.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests~ (live schema comparison below)
- [ ] ~doc.go added or updated~ (no package API change)

## QA steps

Use Bash, an initialized LXD installation, Go and Juju's normal build
dependencies, including the dqlite and musl dependencies in `_deps`.
Start from a checkout containing this PR commit and its upstream 4.1
base. Create separate checkouts and binary directories:

```sh
export ORIGINAL="$PWD"
export DEPS="$PWD/_deps"
export QA_ROOT="$(mktemp -d)"
git --no-pager worktree add --detach "$QA_ROOT/before" d380e5e4fba4c9918a0774638ffa3138aeece73e
git --no-pager worktree add --detach "$QA_ROOT/after" c8282cfab478973d6d92727b2b19c3980f68f84b
ln -s "$DEPS" "$QA_ROOT/before/_deps"
ln -s "$DEPS" "$QA_ROOT/after/_deps"
```

Select upstream 4.1 first:

```sh
export SOURCE="$QA_ROOT/before"
export CONTROLLER=schema-order-before
```

Build and run these operations:

```sh
cd "$SOURCE"
export GOBIN="$QA_ROOT/$CONTROLLER-bin"
mkdir -p "$GOBIN"
make go-install
"$GOBIN/juju" bootstrap lxd "$CONTROLLER" --debug
"$GOBIN/juju" add-model -c "$CONTROLLER" qa
"$GOBIN/juju" deploy -m "$CONTROLLER:qa" ubuntu-lite
"$GOBIN/juju" status -m "$CONTROLLER:qa"
```

Bootstrap must report `No packaged binary found, preparing local Juju
agent binary` and identify `$GOBIN/jujuagentd` as the agent to upload.
If it selects a packaged agent instead, stop: that controller would not
test the intended build. Expect bootstrap and model creation to succeed
and `ubuntu-lite/0` to reach active status.

Capture the actual dqlite schema from the global controller database and
the `qa` model database:

```sh
"$GOBIN/juju" ssh -m "$CONTROLLER:controller" 0 bash > "$QA_ROOT/$CONTROLLER.schema" <<'REMOTE'
. /etc/profile.d/juju-introspection.sh
juju_db_repl <<'SQL'
SELECT 'CONTROLLER DATABASE' AS database_name;
SELECT type, name, tbl_name, sql FROM sqlite_master WHERE name NOT GLOB 'sqlite_stat*' ORDER BY type, name;
.switch model-qa
SELECT 'MODEL DATABASE' AS database_name;
SELECT type, name, tbl_name, sql FROM sqlite_master WHERE name NOT GLOB 'sqlite_stat*' ORDER BY type, name;
.exit
SQL
REMOTE
```

Check that the capture contains both sections: the controller section
includes `v_model_all`, and the model section includes `v_model_metrics`.
Check the REPL output for errors.

Select the patched checkout and repeat the same build, bootstrap, model,
deployment and capture operations:

```sh
export SOURCE="$QA_ROOT/after"
export CONTROLLER=schema-order-after
```

Expect the same functional results. Compare the schema captures:

```sh
diff -u "$QA_ROOT/schema-order-before.schema" "$QA_ROOT/schema-order-after.schema"
```

Expected diff:

```diff
--- schema-order-before.schema	2026-11-18 21:57:24.241065869 -0500
+++ schema-order-after.schema	2026-11-18 21:58:53.346070621 -0500
@@ -8400,7 +8400,7 @@
     END) AS subject_id,
     -- scope_id is the natural id of the scope entity (uuid for model)
     -- relations are not processed here - their scope_id is the relation key
-    -- which needs to be composed in code. 
+    -- which needs to be composed in code.
     (CASE
         WHEN sp.scope_type_id = 0 THEN scu.name
         WHEN sp.scope_type_id = 1 THEN sca.name
```

The definitions should match apart from the removed trailing space in a
moved `v_secret_permission` comment. The query sorts objects by type and name,
so creation order is not part of the comparison. It excludes SQLite statistics
tables and does not compare runtime data or patch hashes.  Investigate any
other schema difference.

Remove the QA controllers and clean source worktrees:

```sh
"$GOBIN/juju" destroy-controller schema-order-after --destroy-all-models --no-prompt
"$GOBIN/juju" destroy-controller schema-order-before --destroy-all-models --no-prompt
cd "$ORIGINAL"
git --no-pager worktree remove "$QA_ROOT/before"
git --no-pager worktree remove "$QA_ROOT/after"
```

## Documentation changes

None.

## Links

None.
