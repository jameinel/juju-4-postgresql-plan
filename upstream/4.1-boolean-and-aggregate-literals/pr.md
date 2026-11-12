Use boolean literals when filtering charm subordination and Kubernetes
controller addresses. These values are booleans, even though SQLite also
accepts comparisons with integers.

Aggregate the machine instance ID alongside the password-match count. The
cloud-instance primary key permits at most one row per machine, so `MAX`
preserves the existing result, including NULL and empty instance IDs.
Compare the controller ID with a text literal in the corresponding test.

The network binding query still contains `IFNULL`; its separate cleanup is
needed before the entire query uses portable SQL.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

On a Linux host with LXD initialized, nested containers enabled, and access
to Charmhub, build the client from this branch and bootstrap a controller with
an agent built from the same source:

```sh
mkdir -p "$PWD/_build/qa-bin"
GOBIN="$PWD/_build/qa-bin" make juju
export PATH="$PWD/_build/qa-bin:$PATH"
juju bootstrap lxd u20-qa --build-agent
juju add-model u20-qa
juju deploy ubuntu-lite
juju status --relations
```

Wait until `ubuntu-lite/0` is active and its machine agent is idle. Add a
unit in a nested container on the first application machine:

```sh
juju add-unit ubuntu-lite --to lxd:0
juju status --relations
juju debug-log -m u20-qa:controller --replay --no-tail
juju debug-log -m u20-qa:u20-qa --replay --no-tail
```

The new unit should become active with an idle agent. The controller and
model logs should not report password/nonce authentication or container
network binding query errors. The controller should remain reachable.

Kubernetes controller-address behavior also needs functional QA with a
controller image built from this branch; `--build-agent` is not supported
for Kubernetes bootstrap.

## Documentation changes

~No user-facing workflow changes.~

## Links

~No issue or specification link.~
