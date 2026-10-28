# fix(schema): split application exposure by scope

Four model tables have nullable columns in a composite primary key. Standard
SQL makes every primary key column `NOT NULL`, so other databases reject
rows that store NULL there. SQLite allows the NULL for backwards
compatibility and treats each NULL as distinct, so the key does not prevent
duplicate rows. The guard test added here lists them when run against `4.1`:

| Table | Nullable key columns | What a NULL means |
|---|---|---|
| `application_exposed_endpoint_space` | `application_endpoint_uuid` | All endpoints (the `""` wildcard) |
| `application_exposed_endpoint_cidr` | `application_endpoint_uuid` | All endpoints (the `""` wildcard) |
| `charm_container_mount` | `charm_container_key` | Nothing; Juju always writes the key |
| `charm_manifest_base` | `os_id`, `track`, `branch`, `architecture_id` | An unset attribute of the base |

The controller schema has none.

### `fix(schema): require keys for charm container mounts and manifest bases`

- `charm_container_mount.charm_container_key` is `NOT NULL`. It is also the
  foreign key to `charm_container`, which a NULL bypassed.
- `charm_manifest_base` is keyed on `(charm_uuid, array_index,
  nested_array_index)`, the manifest position that the encoder already makes
  unique. The descriptive columns remain nullable attributes.
- The 4.0.12 to 4.1.0 import delta rejects a mount without a container key,
  reporting the mount index and charm UUID.

### `fix(schema): split application exposure by scope`

Exposure on all endpoints moves into two application-wide tables. This
mirrors bindings, where `application.space_uuid` holds the default binding
and `application_endpoint.space_uuid` holds each endpoint's binding: an
application is exposed in the same shape as it is bound.

| Scope | Spaces | CIDRs |
|---|---|---|
| All endpoints (new tables) | `application_exposed_space`, key `(application_uuid, space_uuid)` | `application_exposed_cidr`, key `(application_uuid, cidr)` |
| One endpoint | `application_exposed_endpoint_space`, endpoint now `NOT NULL` | `application_exposed_endpoint_cidr`, endpoint now `NOT NULL` |

- `v_application_exposed_endpoint` combines all four tables and still
  reports application-wide rows with a NULL endpoint. Readers, the API and
  the `""` wildcard endpoint are unchanged.
- The expose writers store the wildcard in the new tables. `juju unexpose
  <app>` clears all four.
- `WatchApplicationExposed` watches all four tables, using new change-log
  triggers.
- Application removal deletes rows from the new tables.
- Space removal:
  - `juju remove-space` without `--force` still refuses when an
    application is exposed to the space on all endpoints.
  - With `--force`, exposure moves to `alpha`. If the application or
    endpoint is already exposed to `alpha`, the row for the removed space
    is deleted instead of moved.
  - This fixes a pre-existing bug. `remove-space --force` failed with a
    UNIQUE constraint error when an endpoint was exposed to both `alpha`
    and the removed space. For all-endpoint exposure it silently created
    duplicate rows.
- The 4.1.0 export types are regenerated. The `to_v4_1_0` deltas split
  4.0.12 rows by scope and collapse duplicate wildcard rows that the old
  key allowed.
- `TestPrimaryKeySchemaSuite` checks the model and controller schemas for
  nullable primary key columns.

### Schema change policy

The 4.1 base schema is edited in place, as in juju/juju#23690 and
juju/juju#23682. 4.1 is in beta, the 4.1.0 payload is unreleased, and no
post-patch is registered. Controllers bootstrapped from an earlier 4.1 beta
cannot be upgraded in place. 4.0 models reach 4.1 by migration, which
applies the deltas.

### Out of scope

- `port_range.relation_uuid` also uses NULL as a wildcard, but it is not
  part of a primary key.
- A 4.0.12 payload with duplicate manifest positions or a NULL container
  key fails import. Juju does not write either.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~ (covered by the manual QA below; the existing firewall expose tests run on AWS only)
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

### Unit tests

```sh
go test -race ./domain/schema/... ./domain/application/... ./domain/network/... \
    ./domain/removal/... ./domain/modelimport/... ./domain/recovery/...
```

### Setup

Build the branch and make sure its `juju` is first on `PATH`.

```sh
make install
juju version   # 4.1-beta…
juju bootstrap lxd qa41 --build-agent
juju add-model expose41
juju deploy ubuntu-lite
juju add-space qa-space
juju add-space qa-gone
```

Wait for `ubuntu-lite/0` to be active. The order of spaces and CIDRs in
`show-application` output may vary.

### 1. Expose on all endpoints

```sh
juju expose ubuntu-lite --to-spaces alpha,qa-gone --to-cidrs 10.0.0.0/24
juju show-application ubuntu-lite
juju remove-space qa-gone
```

- `exposed-endpoints` has a `""` entry with spaces `alpha` and `qa-gone`,
  and CIDR `10.0.0.0/24`.
- `remove-space` is refused with
  `- "qa-gone" is used as a binding on: ubuntu-lite`.

### 2. Expose one endpoint

```sh
juju expose ubuntu-lite --endpoints ubuntu \
    --to-spaces alpha,qa-gone,qa-space --to-cidrs 10.42.0.0/16
juju show-application ubuntu-lite
```

- The `""` entry is unchanged.
- A new `ubuntu` entry has spaces `alpha`, `qa-gone` and `qa-space`, and
  CIDR `10.42.0.0/16`.

### 3. Force space removal

Both entries are exposed to `alpha` as well as `qa-gone`.

```sh
juju remove-space qa-gone --force -y
juju show-application ubuntu-lite
```

- The command succeeds.
- `""` has space `alpha` and CIDR `10.0.0.0/24`.
- `ubuntu` has spaces `alpha` and `qa-space`, and CIDR `10.42.0.0/16`.

Without this change, the command fails with a UNIQUE constraint error,
because the `ubuntu` entry's `qa-gone` row cannot move to `alpha`.

### 4. Unexpose and remove

```sh
juju unexpose ubuntu-lite --endpoints ubuntu
juju show-application ubuntu-lite   # exposed: true, only "" remains
juju unexpose ubuntu-lite
juju show-application ubuntu-lite   # exposed: false
juju expose ubuntu-lite --to-spaces qa-space
juju remove-application ubuntu-lite --no-prompt
juju status                         # repeat until ubuntu-lite is gone
```

The application is removed. Any exposure rows left behind would fail the
deletion on their foreign keys.

### 5. 4.0 to 4.1 migration

Use a 4.0 client and controller, for example from the `4.0/stable` snap
(`snap refresh` it if the snap is already installed):

```sh
sudo snap install juju --channel 4.0/stable
J40=/snap/bin/juju
$J40 bootstrap lxd src40
$J40 add-model expose40
$J40 deploy ubuntu-lite
$J40 add-space qa-space
$J40 add-space qa-gone
$J40 expose ubuntu-lite --to-spaces alpha,qa-gone --to-cidrs 10.0.0.0/24
$J40 expose ubuntu-lite --endpoints ubuntu --to-spaces qa-space --to-cidrs 10.42.0.0/16
$J40 remove-space qa-gone --force -y
```

On 4.0, the force removal moves the all-endpoints `qa-gone` row to `alpha`,
so the model stores two all-endpoints `alpha` rows.

Wait for `ubuntu-lite/0` to be active, then migrate the model to the
branch controller:

```sh
$J40 migrate src40:expose40 qa41
juju models -c qa41   # repeat until expose40 appears
juju show-application -m qa41:expose40 ubuntu-lite
juju add-unit -m qa41:expose40 ubuntu-lite
```

- The migration completes. The new tables reject duplicate rows, so this
  shows the import collapsed the two `alpha` rows.
- `""` has space `alpha` and CIDR `10.0.0.0/24`.
- `ubuntu` has space `qa-space` and CIDR `10.42.0.0/16`.
- The new unit becomes active.

### Clean up

```sh
juju destroy-controller qa41 --destroy-all-models --no-prompt
$J40 destroy-controller src40 --destroy-all-models --no-prompt
```

## Documentation changes

None. CLI and API behaviour is unchanged, except that `juju remove-space
--force` no longer fails when an endpoint is exposed to both the removed
space and `alpha`.

## Links

~**Jira card:**~ None.
