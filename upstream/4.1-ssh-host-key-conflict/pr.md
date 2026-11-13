An expected virtual SSH host-key uniqueness violation is followed by a read
in the same transaction. On databases that abort a transaction after a failed
statement, that read cannot return the existing key.

Use targeted `ON CONFLICT DO NOTHING` inserts for machine and unit host keys,
then read the persisted key in the same transaction. Concurrent creators
return the winning key without replacing its algorithm; unrelated constraint
errors, read errors and transaction retry errors remain visible. Keep the two
entity-specific paths explicit rather than parameterising their SQL and record
types through a shared helper.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with descriptive test names
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

On a host with LXD, a usable local SSH key, and the dependencies for building
Juju, build this checkout's client and bootstrap a disposable controller:

```sh
export QA_DIR="$(mktemp -d)"
chmod 700 "$QA_DIR"
mkdir "$QA_DIR/bin"
export GOBIN="$QA_DIR/bin"
export PATH="$GOBIN:$PATH"
make go-install
juju bootstrap lxd ssh-qa --build-agent
juju add-model -c ssh-qa machine-keys
juju add-ssh-key -m ssh-qa:machine-keys "$(cat ~/.ssh/id_ed25519.pub)"
juju deploy -m ssh-qa:machine-keys juju-qa-test ssh-test --base ubuntu@22.04
juju status -m ssh-qa:machine-keys --watch 5s
```

Wait until `ssh-test/0` is active and idle, then stop the status watch.
Run two proxied connections concurrently to the unit, followed by connections
to the unit and its machine:

```sh
juju ssh -m ssh-qa:machine-keys ssh-test/0 -- printf 'first\n' &
p1=$!
juju ssh -m ssh-qa:machine-keys ssh-test/0 -- printf 'second\n' &
p2=$!
wait "$p1"; r1=$?
wait "$p2"; r2=$?
printf 'exit statuses: %s %s\n' "$r1" "$r2"
juju ssh -m ssh-qa:machine-keys ssh-test/0 -- printf 'unit-ok\n'
juju ssh -m ssh-qa:machine-keys 0 -- printf 'machine-ok\n'
```

Both concurrent exit statuses must be zero, and all four commands must print
their markers without a host-key mismatch. Machine-backed units share the
machine key. The concurrent commands exercise competing requests, but do not
guarantee that both reached the first-creation race.

Remove the disposable controller after checking:

```sh
juju destroy-controller ssh-qa --destroy-all-models --destroy-storage
```

For the separate Kubernetes unit-key path, use a configured MicroK8s cloud
and the same client and SSH key:

```sh
juju bootstrap microk8s ssh-k8s-qa --build-agent
juju add-model -c ssh-k8s-qa unit-keys
juju add-ssh-key -m ssh-k8s-qa:unit-keys "$(cat ~/.ssh/id_ed25519.pub)"
juju deploy -m ssh-k8s-qa:unit-keys snappass-test ssh-test
juju status -m ssh-k8s-qa:unit-keys --watch 5s
```

Wait for `ssh-test/0` to be active and idle, then stop the watch and run:

```sh
juju ssh -m ssh-k8s-qa:unit-keys ssh-test/0 -- printf 'first\n' &
p1=$!
juju ssh -m ssh-k8s-qa:unit-keys ssh-test/0 -- printf 'second\n' &
p2=$!
wait "$p1"; r1=$?
wait "$p2"; r2=$?
printf 'exit statuses: %s %s\n' "$r1" "$r2"
juju ssh -m ssh-k8s-qa:unit-keys ssh-test/0 -- printf 'unit-ok\n'
```

Both concurrent exit statuses must be zero, and all three commands must print
their markers without a host-key mismatch. As above, concurrency does not
guarantee that the first-creation race occurred. Clean up the controller:

```sh
juju destroy-controller ssh-k8s-qa --destroy-all-models --destroy-storage
```

## Documentation changes

No user-facing workflow changes.

## Links

No associated issue.
