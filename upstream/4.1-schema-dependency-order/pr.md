SQLite accepts schema definitions that refer to tables or views created
later. This makes the source order misleading and prevents consumers that
require referenced objects to exist from applying the definitions in order.

Order the controller and model schema definitions so foreign-key targets
and their keys exist before the referencing table, and view sources exist
before the view. Split out shared prerequisites where whole-file ordering
cannot satisfy the dependency graph. Preserve the existing definitions,
seed data and generated triggers.

Add a schema-order guard that checks table keys, view dependencies and
trigger dependencies, including references within a single patch. It uses
SQLite's schema metadata and resolver, not PostgreSQL-specific SQL or
runtime dependencies.

This changes the 4.1 beta base-schema hash chain. Earlier 4.1 beta
databases cannot upgrade in place across the change. Released 4.0 models
continue to reach 4.1 through migration; historical release DDL is unchanged.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests~ (functional bootstrap and migration QA below)
- [ ] ~doc.go added or updated~ (no package API change)

## QA steps

Use an initialized LXD installation and a separate Juju 4.0 client. Run
from this checkout with an available Go toolchain and the normal Juju
build dependencies. The two clients must share their Juju client data so
the destination client can find the source controller.

Build and check the clients:

```sh
make install
export J41="$(go env GOPATH)/bin/juju"
export J40=/snap/bin/juju
"$J41" version
"$J40" version
```

Expect the first client to report 4.1 and the second to report 4.0. Use a
different `J40` path if the 4.0 client is installed elsewhere.

Bootstrap a fresh 4.1 controller from the build:

```sh
"$J41" bootstrap lxd schema-order-41 --build-agent
"$J41" add-model -c schema-order-41 fresh
"$J41" deploy -m schema-order-41:fresh ubuntu-lite
"$J41" status -m schema-order-41:fresh
```

Expect bootstrap and model creation to succeed and `ubuntu-lite/0` to
reach active status. Check the controller and model logs for schema,
foreign-key or trigger errors:

```sh
"$J41" debug-log -m schema-order-41:controller --replay --no-tail --level WARNING
"$J41" debug-log -m schema-order-41:fresh --replay --no-tail --level WARNING
```

Create a model on a 4.0 controller and migrate it to the new controller:

```sh
"$J40" bootstrap lxd schema-order-40
"$J40" add-model -c schema-order-40 moveme
"$J40" deploy -m schema-order-40:moveme ubuntu-lite
"$J40" status -m schema-order-40:moveme
```

Wait for `ubuntu-lite/0` to become active, then migrate and exercise a
new write on the destination:

```sh
"$J41" migrate schema-order-40:moveme schema-order-41
"$J41" add-unit -m schema-order-41:moveme ubuntu-lite
"$J41" status -m schema-order-41:moveme
"$J41" debug-log -m schema-order-41:moveme --replay --no-tail --level WARNING
```

Expect migration to finish, both units to become active and no schema,
foreign-key or trigger errors in the model log. Remove the QA controllers:

```sh
"$J41" destroy-controller schema-order-41 --destroy-all-models --no-prompt
"$J40" destroy-controller schema-order-40 --destroy-all-models --no-prompt
```

## Documentation changes

None.

## Links

None.
