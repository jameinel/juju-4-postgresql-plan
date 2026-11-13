The controller and model agent-store queries project the caller's stream
as an untyped SQL parameter, although binary rows do not store it. Select
the stored version and architecture instead, and share the query and
decoding between the two stores.

The controller lookup returns all stored binaries without a stream argument
or stream field. The model lookup still returns binaries only when the
requested stream matches the model's target stream; the comparison and
binary query remain in one transaction. This preserves existing selection
behavior, including empty results and errors for invalid stored versions.

Neither store records per-binary stream membership. Consequently, controller
upgrade checks may use a controller-store binary regardless of the requested
stream, while the model lookup assumes its stored binaries apply to the
current target stream. Simple streams queries do filter by stream. Correct
per-binary membership would require a separate storage and migration design;
this change does not add that filtering.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- ~[ ] Comments saying why design decisions were made~
- [x] Go unit tests, ~with comments saying what you're testing~
- ~[ ] [Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~
- ~[ ] [doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

On a host with LXD configured and agent streams reachable, build this branch
and bootstrap a disposable controller using its agent:

```sh
make install
juju bootstrap localhost stream-qa --build-agent
juju add-model -c stream-qa stream-test
```

Check the controller's configured stream and target version, then run
read-only upgrade checks using two streams:

```sh
juju model-config -m stream-qa:controller agent-stream
juju model-config -m stream-qa:controller agent-version
juju upgrade-controller -c stream-qa --dry-run --agent-stream released
juju upgrade-controller -c stream-qa --dry-run --agent-stream devel
```

Each check should report an upgrade candidate or that no upgrade is
available, without an agent-store query or version-parsing error. Results
depend on the available binaries and streams. A missing stream or network
error is not a successful check. These commands do not distinguish a local
store hit from a simplestreams fallback or prove stream membership of a
stored binary.

Verify that dry runs leave the controller target unchanged, then clean up:

```sh
juju model-config -m stream-qa:controller agent-version
juju destroy-controller stream-qa --destroy-all-models --destroy-storage -y
```

The target version must match the value recorded before the dry runs.

## Documentation changes

None.

## Links

None.
