# fix(schema): split application exposure by scope

Published after explicit approval on 2026-10-30:
https://github.com/juju/juju/pull/23723
Base: `4.1`; head: `jameinel:4.1-application-exposure-scope` at `4e66f7c449`.
Review fixes, the body below and the approved reply were published after
explicit approval on 2026-10-30; reply: https://github.com/juju/juju/pull/23723#issuecomment-6531441845

Depends on juju/juju#23710 (charm keys); land that PR first. This branch is
stacked on its charm-only local head. Until the parent lands, GitHub's
comparison to `4.1` includes both PRs: review the exposure-only changes at
https://github.com/jameinel/juju/compare/4.1-nullable-primary-keys...4.1-application-exposure-scope
Rebase onto upstream `4.1` after the charm PR lands.
Current upstream base: `8369e6e2a9`; charm parent: `ba21f48ad3`.
Stacking keeps the combined primary-key guard passing without mixing the
two review concerns.

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
- Writers route wildcard rows to the new tables; `deleteAllExposure` clears
  all four transactionally when unexposing an application. Errors name the
  failed table without embedding SQL.
- `WatchApplicationExposed` watches all four tables via change-log triggers.
  Tests exercise each namespace independently, including wildcard CIDRs,
  so deletion events cannot mask missing subscriptions. The empty-namespace
  guard deliberately rejects alternative state implementations that violate
  the documented non-empty namespace contract.
  Application removal clears application-wide rows before deleting the app.
- Space removal still refuses wildcard exposure without `--force`. Forced
  removal moves exposure to `alpha`, deleting the removed-space row instead
  when exposure to `alpha` already exists. This fixes a pre-existing UNIQUE
  failure for endpoint exposure and duplicate wildcard exposure rows.
  Deduplication and reset errors identify the failed exposure table;
  failure-injection tests verify both operations roll back earlier changes.
- Regenerated 4.1.0 export/import/recovery code carries both scopes. Legacy
  4.0.12 deltas split exposure by scope and collapse duplicate wildcard
  rows permitted by the old key; endpoint-specific rows are preserved.
- `TestPrimaryKeySchemaSuite` guards model and controller schemas after
  both PRs. Single-column INT lookup keys remain exempt here; independent
  juju/juju#23724 supplies an INT-key-scoped guard. When the exposure and
  lookup changes are integrated, regenerate from the combined schema and
  remove the global guard's INT exemption, retaining only the INTEGER rowid
  exemption. Run both scoped and global guards.

### Schema policy and scope

Edit the beta-only 4.1 base schema in place, as in juju/juju#23690 and
juju/juju#23682. The 4.1.0 payload is unreleased; no post-patch is registered.
Earlier 4.1-beta controllers cannot upgrade in place across these hash
changes. Once a 4.1 release candidate ships, subsequent schema changes must
use new patches rather than further in-place edits. 4.0 models reach 4.1
via migration and the validated deltas.
Historical release DDL and 4.0 export types stay unchanged.

Charm-key implementation and its generator metadata review suggestion
belong to juju/juju#23710; this follow-up does not change the parent PR's
generator. `port_range`'s nullable relation wildcard is not a primary key
and is outside this PR.

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

Review-follow-up validation also passed:

```sh
make run-go-tests \
  TEST_PACKAGES="./domain/application/service ./domain/application/state ./domain/application ./domain/network/state" \
  TEST_ARGS="-race -count=1 -p=2 -timeout=4m" \
  TEST_FILTER='"TestExposedStateSuite|TestSpaceDeleteSuite|TestApplicationWatcherServiceSuite|TestWatcherSuite/TestWatchApplicationExposed"'
```

This covers the full exposure-state, space-deletion and application-watcher
service suites, plus the exposure watcher integration tests. Focused lint
with `dqlite,libsqlite3` build tags reported zero issues; `make go-build`,
import formatting and diff checks passed. `make pre-check` passed but skips
static analysis by default; the explicit focused lint supplies that check.
The strengthened exposure watcher completed 6 stress runs without failures
before its intentional outer timeout.

Verify both scopes through writes, reads, watcher events, unexpose,
application removal, force space removal, legacy transform/import/export
and recovery. Failure-injection tests verify that errors name the failed
table and preceding changes roll back for all four unexpose deletion tables
and for deduplication and reset of each exposure-space table.

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
