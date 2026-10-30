# fix(schema): split application exposure by scope

Published after explicit approval on 2026-10-30:
https://github.com/juju/juju/pull/23723
Base: `4.1`; head: `jameinel:4.1-application-exposure-scope` at `f31de64e52`.
The title/body match the approved publication; this metadata is local only.

Depends on juju/juju#23710 (charm keys); land that PR first. This branch is
stacked on its charm-only local head. Until the parent lands, GitHub's
comparison to `4.1` includes both PRs: review the exposure-only changes at
https://github.com/jameinel/juju/compare/4.1-nullable-primary-keys...4.1-application-exposure-scope
Rebase onto upstream `4.1` after the charm PR lands.
Current upstream base: `8369e6e2a9`; charm parent: `ba21f48ad3`;
exposure head: `f31de64e52`. Stacking keeps the combined primary-key guard
passing without mixing the two review concerns.

The endpoint exposure tables use a NULL endpoint UUID to mean all
endpoints. SQLite permits NULL inside the composite primary key and treats
NULLs as distinct, allowing duplicate wildcard rows. Standard SQL requires
primary-key columns to be NOT NULL, rejecting valid wildcard exposure.

Move all-endpoint exposure into application-wide tables, mirroring default
versus per-endpoint bindings:

| Scope | Spaces | CIDRs |
|---|---|---|
| All endpoints | `application_exposed_space`, key `(application_uuid, space_uuid)` | `application_exposed_cidr`, key `(application_uuid, cidr)` |
| One endpoint | `application_exposed_endpoint_space`, endpoint NOT NULL | `application_exposed_endpoint_cidr`, endpoint NOT NULL |

- `v_application_exposed_endpoint` combines all four tables and still
  reports application-wide rows with a NULL endpoint. Readers, the API
  and the `""` wildcard endpoint remain unchanged.
- Writers route wildcard rows to the new tables; unexposing an application
  clears all four transactionally. Errors identify the failed query/table.
- `WatchApplicationExposed` watches all four tables via change-log triggers.
  Application removal clears application-wide rows before deleting the app.
- Space removal still refuses wildcard exposure without `--force`. Forced
  removal moves exposure to `alpha`, deleting the removed-space row instead
  when exposure to `alpha` already exists. This fixes a pre-existing UNIQUE
  failure for endpoint exposure and duplicate wildcard exposure rows.
- Regenerated 4.1.0 export/import/recovery code carries both scopes. Legacy
  4.0.12 deltas split exposure by scope and collapse duplicate wildcard
  rows permitted by the old key; endpoint-specific rows are preserved.
- `TestPrimaryKeySchemaSuite` guards model and controller schemas after
  both PRs. Single-column INT lookup keys remain exempt and are a separate
  independent follow-up (`4.1-lookup-key-not-null`). That review will own
  an INT-key-scoped guard; the global guard is tightened when both land.

### Schema policy and scope

Edit the beta-only 4.1 base schema in place, as in juju/juju#23690 and
juju/juju#23682. The 4.1.0 payload is unreleased; no post-patch is registered.
Earlier 4.1-beta controllers cannot upgrade in place across these hash
changes. 4.0 models reach 4.1 via migration and the validated deltas.
Historical release DDL and 4.0 export types stay unchanged.

Charm-key implementation belongs to juju/juju#23710. `port_range`'s nullable
relation wildcard is not a primary key and is outside this PR.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~ (automated domain integration coverage; live QA below remains proposed)
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

### Automated checks

```sh
make rebuild-export
make run-go-tests \
  TEST_PACKAGES="./generate/export ./domain/schema ./domain/application ./domain/application/service ./domain/network/state ./domain/removal/state/model ./domain/modelimport/... ./domain/recovery/..." \
  TEST_ARGS="-race -count=1 -p=2 -timeout=12m" TEST_FILTER=Test
make run-go-tests TEST_PACKAGES="./domain/application/state" \
  TEST_ARGS="-race -count=1 -p=2 -timeout=4m" \
  TEST_FILTER=TestExposedStateSuite
```

At `f31de64e52`, the 13 packages above and the focused exposure-state race
suite passed. Explicit build-tagged focused lint reported zero issues;
regeneration was idempotent. Exposure-watcher stress reported 6 completed
runs and zero failures before its bounded timeout. The refreshed base and
stack both passed `make go-build` for all linux/amd64 targets. The entire
application-state suite was not rerun (the initial pre-refresh broad run
hit its cumulative timeout); live QA below has not been run.

Verify both scopes through writes, reads, watcher events, unexpose,
application removal, force space removal, legacy transform/import/export
and recovery. The failure-injection regression checks the error identifies
each of the four deletion tables and preceding deletions roll back.

### Live QA (proposed, not run)

Build this branch (`make install`) and put its `juju` first on PATH.
Create dedicated QA controllers only; destroy them after this run.

```sh
juju bootstrap lxd qa41 --build-agent
juju add-model expose41
juju deploy ubuntu-lite
juju add-space qa-space
juju add-space qa-gone
```

Wait for `ubuntu-lite/0` to be active. Expose both scopes, including `alpha`
and the space to remove:

```sh
juju expose ubuntu-lite --to-spaces alpha,qa-gone --to-cidrs 10.0.0.0/24
juju expose ubuntu-lite --endpoints ubuntu \
  --to-spaces alpha,qa-gone,qa-space --to-cidrs 10.42.0.0/16
juju show-application ubuntu-lite
juju remove-space qa-gone
```

Expect a `""` wildcard entry and an `ubuntu` entry with the specified
spaces/CIDRs. Unforced space removal is refused because the space is used.

```sh
juju remove-space qa-gone --force -y
juju show-application ubuntu-lite
juju unexpose ubuntu-lite --endpoints ubuntu
juju show-application ubuntu-lite
juju unexpose ubuntu-lite
juju show-application ubuntu-lite
juju expose ubuntu-lite --to-spaces qa-space
juju remove-application ubuntu-lite --no-prompt
```

Forced removal succeeds: wildcard has only `alpha`; endpoint has `alpha`
and `qa-space`; CIDRs remain unchanged. Endpoint unexpose keeps wildcard;
full unexpose removes all exposure. Application removal finishes without
foreign-key errors.

### Legacy migration (proposed, not run)

Use a separate 4.0 client (`J40=/snap/bin/juju` if the snap is 4.0/stable)
to bootstrap `src40`, deploy `ubuntu-lite` in model `expose40`, and create
`qa-space` and `qa-gone`. Expose the wildcard to `alpha,qa-gone` and CIDR
`10.0.0.0/24`; expose endpoint `ubuntu` to `qa-space` and CIDR
`10.42.0.0/16`. On 4.0, removing `qa-gone --force -y` creates duplicate
wildcard `alpha` rows. After the unit is active:

```sh
$J40 migrate src40:expose40 qa41
juju models -c qa41
juju show-application -m qa41:expose40 ubuntu-lite
juju add-unit -m qa41:expose40 ubuntu-lite
```

Migration finishes; wildcard has one `alpha` exposure and its CIDR;
`ubuntu` retains `qa-space` and its CIDR. The new unit becomes active.

```sh
juju destroy-controller qa41 --destroy-all-models --no-prompt
$J40 destroy-controller src40 --destroy-all-models --no-prompt
```

## Documentation changes

No CLI/API workflow change, except that `remove-space --force` succeeds
when an endpoint is already exposed to both the removed space and `alpha`.

## Links

**Dependency:** juju/juju#23710.
~**Jira card:**~ None.
