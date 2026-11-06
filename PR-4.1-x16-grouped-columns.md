# fix(secretbackend): preserve cloud certificates in portable queries

Secret backend queries select ungrouped columns alongside aggregates or use
GROUP BY to collapse join rows. These queries depend on permissive SQLite
semantics, and a cloud can have more than one CA certificate for a model.
The backend configuration previously retained only an unspecified certificate.

Group every non-aggregate projection in the counting queries and use DISTINCT
for the non-aggregate Kubernetes lookups. Preserve all cloud CA certificates
in both backend consumers, deduplicating repeated join rows and returning a
stable certificate order. Secret counts still count distinct secrets rather
than revisions.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [ ] ~Comments saying why design decisions were made~
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~Integration tests, with comments saying what you're testing~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

Use a disposable Ubuntu host with initialized LXD, Go build prerequisites,
`jq`, `openssl`, and permission to use LXD and sudo. Install MicroK8s on this
host, or use a disposable MicroK8s cluster already installed there:

```sh
sudo snap install microk8s --classic
sudo microk8s status --wait-ready
sudo microk8s enable dns
sudo microk8s enable hostpath-storage
```

In a separate shell, create a private temporary directory and install the
client and agent there, without changing your normal Go bin directory. From
this checkout, bootstrap a disposable LXD controller. `--build-agent` ensures
the controller receives the agent from this checkout rather than a published
agent:

```sh
export QA_DIR="$(mktemp -d)"
chmod 700 "$QA_DIR"
mkdir "$QA_DIR/bin"
export GOBIN="$QA_DIR/bin"
export PATH="$GOBIN:$PATH"
make go-install
juju bootstrap --build-agent lxd qa-secret-controller
```

Register MicroK8s on the client only first. The controller machine must be
able to reach the API endpoint printed by `microk8s config`; do not use its
loopback address. The temporary kubeconfig contains credentials and must
remain private.

```sh
sudo microk8s config > "$QA_DIR/kubeconfig"
chmod 600 "$QA_DIR/kubeconfig"
export KUBECONFIG="$QA_DIR/kubeconfig"
juju add-k8s qa-k8s --client --cloud lxd
juju show-cloud qa-k8s --client --format json > "$QA_DIR/cloud-original.json"
jq -r '.[0].endpoint' "$QA_DIR/cloud-original.json"
jq -e '.[0] | .type == "kubernetes" and (."ca-credentials" | length == 1)' \
  "$QA_DIR/cloud-original.json"
```

Generate a second, unrelated CA certificate; keep the cluster's original CA
so TLS to the Kubernetes API continues to work. Build a cloud definition from
the locally registered cloud and update the client. The generated JSON is also
valid YAML for `update-k8s`. Then register that two-certificate cloud and its
credential with the controller.

```sh
openssl req -x509 -newkey rsa:2048 -nodes -days 1 \
  -subj '/CN=juju-qa-extra-ca' -addext 'basicConstraints=critical,CA:TRUE' \
  -keyout "$QA_DIR/extra-ca.key" -out "$QA_DIR/extra-ca.crt"
jq -e --rawfile extra "$QA_DIR/extra-ca.crt" '
  .[0] | select(.type == "kubernetes" and (."ca-credentials" | length) == 1) |
  {clouds: {"qa-k8s": (
    {type, "auth-types": ."auth-types", endpoint, regions, config,
     "region-config": ."region-config", "skip-tls-verify": ."skip-tls-verify",
     "host-cloud-region": "lxd",
     "ca-certificates": (."ca-credentials" + [$extra])}
    | with_entries(select(.value != null))
  )}}
' "$QA_DIR/cloud-original.json" > "$QA_DIR/cloud.json"
juju update-k8s qa-k8s --client -f "$QA_DIR/cloud.json"
juju add-cloud --controller qa-secret-controller qa-k8s
juju show-cloud qa-k8s --controller qa-secret-controller --format json |
  jq -e '.[0]."ca-credentials" | length == 2'
```

Create a Kubernetes model and two secrets, updating one to make a second
revision. Inspect the built-in backend:

```sh
juju add-model qa-secrets qa-k8s
juju model-secret-backend auto
juju model-secret-backend
juju add-secret qa-first value=one
juju update-secret qa-first value=two
QA_SECOND="$(juju add-secret qa-second value=three)"
juju secret-backends --format json > "$QA_DIR/backends.json"
jq -e '."qa-secrets-local".secrets == 2' "$QA_DIR/backends.json"
jq -e --slurpfile cloud "$QA_DIR/cloud.json" '
  (."qa-secrets-local".config."ca-certs" | sort) ==
  ($cloud[0].clouds."qa-k8s"."ca-certificates" | sort)
' "$QA_DIR/backends.json"
juju show-secret-backend qa-secrets-local
```

`juju model-secret-backend` must return `auto`, which selects the built-in
Kubernetes backend for this model. Both `jq` checks must return `true`: the
count is two secrets, not three revisions, and both certificates appear
exactly once. The backend commands must succeed without a query error.
Verify removal of the second user secret from Kubernetes. This exercises the
removal service's separate backend lookup and its use of the certificate
bundle. The listing comparison above directly checks that both certificates
are returned; a cleanup failure alone does not establish its cause.

```sh
sudo microk8s kubectl -n qa-secrets get secret "${QA_SECOND##*:}-1"
juju remove-secret "$QA_SECOND"
sudo microk8s kubectl -n qa-secrets get secret "${QA_SECOND##*:}-1" \
  --ignore-not-found -o name
juju debug-log -m qa-secrets --replay --no-tail
```

The first command must find the Kubernetes Secret. Removal is asynchronous:
repeat the second command until it produces no output. Check the model log for
secret backend initialization or content deletion errors. Finally remove the
model and disposable controller:

```sh
juju destroy-model qa-secrets --no-prompt
juju models
juju destroy-controller qa-secret-controller --destroy-all-models --no-prompt
```

Wait for model removal before checking `juju models`.
Destroying the disposable controller cleans up the remaining test resources.
Remove the local cloud and temporary files, including the test credential:

```sh
juju remove-k8s qa-k8s --client
rm "$QA_DIR/kubeconfig" "$QA_DIR/cloud-original.json" "$QA_DIR/extra-ca.key" \
  "$QA_DIR/extra-ca.crt" "$QA_DIR/cloud.json" "$QA_DIR/backends.json"
rm "$GOBIN/juju" "$GOBIN/jujuc" "$GOBIN/containeragent" \
  "$GOBIN/juju-metadata" "$GOBIN/jujuagentd" "$GOBIN/pebble"
rmdir "$GOBIN"
rmdir "$QA_DIR"
unset KUBECONFIG GOBIN QA_DIR QA_SECOND
```

## Documentation changes

None.

## Links

None.
