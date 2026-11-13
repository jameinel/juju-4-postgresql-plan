`make install-dependencies` omits tools used for builds, tests and static
analysis. It also uses the patch version from `go.mod` as the Go snap
channel, although snap tracks use the major and minor version.

Install the missing system packages and Go development tools, select the
correct Go snap track and compare Go versions in version order. Preserve
Go tools already on `PATH` and add the configured Go binary directory to
the static-analysis `PATH`.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [ ] ~Comments saying why design decisions were made~
- [ ] ~Go unit tests, with comments saying what you're testing~
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

On a host with LXD configured and network access, create a fresh Ubuntu
container and mount a checkout of this branch read-only:

```sh
git clone --branch 4.1-install-dependencies \
  https://github.com/jameinel/juju.git juju-deps-qa
cd juju-deps-qa
lxc launch ubuntu:24.04 juju-deps-qa -c security.nesting=true
lxc exec juju-deps-qa -- cloud-init status --wait
lxc config device add juju-deps-qa checkout disk \
  source="$PWD" path=/home/ubuntu/juju readonly=true
lxc exec juju-deps-qa -- apt-get update
lxc exec juju-deps-qa -- apt-get install -y make
```

Run the dependency target as the container's Ubuntu user:

```sh
lxc exec juju-deps-qa --user 1000 --group 1000 \
  --env HOME=/home/ubuntu --cwd /home/ubuntu/juju -- \
  make install-dependencies
```

The target should install Go, system packages and the missing Go tools,
then complete successfully. It prints the Go binary directory to add to
the shell's `PATH`.

Check the installed tools:

```sh
lxc exec juju-deps-qa --user 1000 --group 1000 \
  --env HOME=/home/ubuntu --cwd /home/ubuntu/juju -- bash -c '
    export PATH="$PATH:$HOME/go/bin"
    for tool in gcc jq shellcheck yq golangci-lint gci stress shfmt \
      go-licenses govulncheck; do
      command -v "$tool" || exit 1
    done
    golangci-lint --version
    yq --version
  '
```

Each tool should be found, golangci-lint should report version 2.14.x,
and yq should report version 4. Run the setup again and check that it
does not replace the Go tools:

```sh
lxc exec juju-deps-qa --user 1000 --group 1000 \
  --env HOME=/home/ubuntu --cwd /home/ubuntu/juju -- bash -c '
    set -e
    before=$(sha256sum "$HOME"/go/bin/{golangci-lint,gci,stress,shfmt,go-licenses,govulncheck})
    make install-dependencies
    after=$(sha256sum "$HOME"/go/bin/{golangci-lint,gci,stress,shfmt,go-licenses,govulncheck})
    test "$before" = "$after"
    echo "Go tool binaries unchanged"
  '
```

The second run should succeed and print `Go tool binaries unchanged`.
Remove the disposable container:

```sh
lxc config device remove juju-deps-qa checkout
lxc delete --force juju-deps-qa
```

## Documentation changes

~None.~

## Links

~None.~
