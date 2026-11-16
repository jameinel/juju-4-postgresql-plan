Ingress and egress CIDR getters relied on incidental index order. Explicitly
order stored CIDR strings and document lexical ordering at the state and
service boundaries. This is string ordering, not numeric address ordering.
Relation filtering and lifecycle behavior are unchanged.

The egress getter feeds watcher updates for relation-specific CIDRs. The
watcher's separate initial query and its model-config and unit-address
fallbacks do not acquire an ordering guarantee. The offering firewaller
treats ingress CIDRs as a set.

Maintainers: was sorted ingress output intended as a contract, or was the
existing test capturing incidental SQLite ordering? This change proposes
lexical ordering for both getters; it does not assume historical intent.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

Use two disposable LXD controllers and isolated models. The database write
below inserts a synthetic egress range and can change firewall rules; do not
run it on a production controller. This setup offers the provider endpoint
and consumes it from a requirer, so both relevant firewaller workers run.

Build the client and agent from this branch:

```sh
export GOBIN="$(mktemp -d)"
export PATH="$GOBIN:$PATH"
make go-install
```

Create the provider offer on the first controller, and add your ssh key:

```sh
juju bootstrap lxd x17-offer
juju add-ssh-key -m controller "$(cat ~/.ssh/id_ed25519.pub)"
juju add-model offer
juju model-config 'logging-config=<root>=INFO;juju.worker.firewaller=TRACE'
juju deploy juju-qa-dummy-sink --base ubuntu@22.04
juju offer dummy-sink:source dummy-offer
```

Create the consuming requirer on the second controller:

```sh
juju bootstrap lxd x17-consume
juju add-ssh-key -m controller "$(cat ~/.ssh/id_ed25519.pub)"
juju add-model consume
juju model-config 'logging-config=<root>=INFO;juju.worker.firewaller=TRACE'
juju deploy juju-qa-dummy-source --base ubuntu@22.04 --config token=becomegreen
juju consume x17-offer:admin/offer.dummy-offer
juju integrate dummy-source dummy-offer --via 10.2.0.0/16
```

Wait for both applications to settle and the offer to show a connected
consumer. The offer should show `dummy-sink:source` as a provider and the
integration should show `dummy-source:sink` as a requirer. Check that the
consuming model has an egress row for `10.2.0.0/16`:

```sh
watch --color "juju status -m x17-offer:offer --color; juju status -m x17-consume:consume --color"
```

Stop `watch` once the relation is connected. SSH to the consuming controller
and start an interactive REPL:

```sh
juju ssh -m x17-consume:controller 0
. /etc/profile.d/juju-introspection.sh
juju_db_repl
```

At the REPL prompt, run:

```sql
.switch model-consume
SELECT relation_uuid, cidr FROM relation_network_egress ORDER BY relation_uuid, cidr;
```

Check the SELECT output and any preceding query errors. At the REPL prompt,
press Ctrl-C to exit, then exit the SSH session:

With only this one relation in the disposable consuming model, add a second
CIDR to its existing egress row. This triggers a watcher update, rather than
relying on the separately implemented initial query. The follow-up SELECT
should show both CIDRs for the same relation. SSH to the controller and start
the REPL again:

```sh
juju ssh -m x17-consume:controller 0
. /etc/profile.d/juju-introspection.sh
juju_db_repl
```

At the REPL prompt, run:

```sql
.switch model-consume
INSERT INTO relation_network_egress (relation_uuid, cidr) SELECT relation_uuid, '10.10.0.0/16' FROM relation_network_egress WHERE cidr = '10.2.0.0/16';
SELECT relation_uuid, cidr FROM relation_network_egress ORDER BY relation_uuid, cidr;
```

From the local shell, check the firewaller logs:

```sh
juju debug-log -m x17-consume:consume --replay --no-tail --include-module juju.worker.firewaller | grep 'local egress addresses'
juju debug-log -m x17-offer:offer --replay --no-tail --include-module juju.worker.firewaller | grep 'offerer relation ingress addresses'
```

After both watchers process the update, the consuming egress and offering
ingress log lines should contain `10.10.0.0/16` before `10.2.0.0/16`. The
relation should remain connected. If necessary, repeat the log commands while
waiting for the watchers.

If either log is empty, inspect the unfiltered firewaller logs for worker
errors instead of treating the empty output as a pass:

```sh
juju debug-log -m x17-consume:consume --replay --no-tail --include-module juju.worker.firewaller
juju debug-log -m x17-offer:offer --replay --no-tail --include-module juju.worker.firewaller
```

Remove the disposable controllers and their models after checking:

```sh
juju destroy-controller x17-consume --destroy-all-models --destroy-storage -y
juju destroy-controller x17-offer --destroy-all-models --destroy-storage -y
rm -rf $GOBIN
```

## Documentation changes

The state and service interface comments specify lexical CIDR ordering; the
ingress service method documents the same contract.

## Links

~No issue or Jira card.~
