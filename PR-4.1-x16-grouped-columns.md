# fix(secretbackend): preserve cloud certificates in portable queries

Sorry for the complex QA steps: exercising this change requires a Kubernetes
model, a reachable API endpoint, and a cloud with multiple CA certificates.

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

**What to check:** The two `jq` checks on `juju secret-backends` are the key
regression checks. The first verifies that two secrets with three revisions
count as two secrets. The second compares the backend's CA certificates with
the two in the cloud definition: before this change the backend could retain
only one. The removal check exercises the separate removal lookup and verifies
that secret content is deleted, but does not independently prove that lookup
returned both certificates. This LXD controller uses dqlite, so this QA does
not itself establish PostgreSQL compatibility.

Use a disposable Ubuntu host with initialized LXD, Go build prerequisites,
`jq`, `openssl`, and permission to use LXD and sudo. Install Canonical
Kubernetes, then bootstrap a disposable cluster:

```sh
sudo snap install k8s --classic
sudo k8s bootstrap
sudo k8s enable network
sudo k8s enable dns
sudo k8s enable local-storage
sudo k8s status --wait-ready
sudo k8s kubectl get nodes
sudo k8s kubectl get storageclass
```

The node must be Ready and local storage available. If the cluster is already
bootstrapped, do not bootstrap it again; use a disposable cluster with network,
DNS, and local storage enabled.

In a separate shell, install the client and agent in a private temporary
directory, without changing your normal Go bin directory. From this checkout,
bootstrap a disposable LXD controller. `--build-agent` ensures the
controller receives the agent from this checkout:

```sh
export QA_DIR="$(mktemp -d)"
chmod 700 "$QA_DIR"
mkdir "$QA_DIR/bin"
export GOBIN="$QA_DIR/bin"
export PATH="$GOBIN:$PATH"
make go-install
juju bootstrap --build-agent lxd qa-secret-controller
```

Export the cluster kubeconfig into the private directory. The kubeconfig
contains credentials and must remain private. Register the cloud locally:

```sh
sudo k8s kubectl config view --raw > "$QA_DIR/kubeconfig"
chmod 600 "$QA_DIR/kubeconfig"
export KUBECONFIG="$QA_DIR/kubeconfig"
sudo k8s kubectl --kubeconfig "$QA_DIR/kubeconfig" get nodes
cat "$QA_DIR/kubeconfig" | juju add-k8s qa-k8s --client --cloud lxd
juju show-cloud qa-k8s --client --format json > "$QA_DIR/cloud-original.json"
jq -r '.[0].endpoint' "$QA_DIR/cloud-original.json"
jq -e '.[0] | .type == "k8s" and (."ca-credentials" | length == 1)' \
  "$QA_DIR/cloud-original.json"
```

The kubeconfig may point to `127.0.0.1`, which refers to the controller
container itself when used from LXD. Find the Kubernetes node's internal IP
and use that address instead:

```sh
sudo k8s kubectl get nodes -o wide
export QA_API_IP="<node-internal-ip>"
export QA_API_ENDPOINT="https://$QA_API_IP:6443"
sudo k8s kubectl --server="$QA_API_ENDPOINT" get nodes
```

The last command must succeed with TLS verification enabled. If it reports a
certificate name mismatch, select an address in the API server certificate
or configure an additional certificate SAN for the cluster; do not disable
TLS verification.

Add an SSH public key to the controller model before using `juju scp` or
`juju ssh`. Replace the path with the public key for an SSH identity
available to your client:

```sh
juju add-ssh-key -m qa-secret-controller:controller "$(cat ~/.ssh/id_ed25519.pub)"
juju ssh -m qa-secret-controller:controller 0 true
```

Wait for the key to propagate and retry the SSH check if needed. Then check
connectivity and TLS from the disposable controller container. An HTTP status
such as 200, 401, or 403 shows that the API was reached; a connection or
certificate error does not:

```sh
jq -r '.[0]."ca-credentials"[0]' "$QA_DIR/cloud-original.json" \
  > "$QA_DIR/cluster-ca.crt"
juju scp -m qa-secret-controller:controller "$QA_DIR/cluster-ca.crt" \
  0:/tmp/x16-qa-k8s-ca.crt
juju ssh -m qa-secret-controller:controller 0 \
  "curl --connect-timeout 5 --cacert /tmp/x16-qa-k8s-ca.crt -sS -o /dev/null -w '%{http_code}\n' '$QA_API_ENDPOINT/version'"
```

If `curl` returns an HTTP status, including 401, the connectivity check passed.
Skip the firewall steps and continue to certificate generation below.

Only if `curl` fails to connect, check the route and the Kubernetes host's
firewall. For an active UFW firewall, find the controller container's IP with
the command below and, if appropriate for the host, allow only that source
to reach the selected API address on TCP port 6443. If UFW is inactive, do not
add a UFW rule; check the route and any other firewall instead:

```sh
juju ssh -m qa-secret-controller:controller 0 'hostname -I'
export QA_CONTROLLER_IP="<controller-container-ip>"
sudo ufw status
sudo ufw allow from "$QA_CONTROLLER_IP" to "$QA_API_IP" port 6443 proto tcp
```

Retry the controller `curl` check. Other firewall implementations require
their own equivalent rule; do not change unrelated firewall settings.
If you added the UFW rule for QA, remove it during cleanup:

```sh
sudo ufw delete allow from "$QA_CONTROLLER_IP" to "$QA_API_IP" port 6443 proto tcp
```

Generate a second, unrelated CA certificate; keep the cluster's original CA
so TLS to the Kubernetes API continues to work. Build a cloud definition from
the locally registered cloud, using the reachable endpoint, and update the
client. The generated JSON is also valid YAML for `update-k8s`. Then register
that two-certificate cloud and its credential with the controller.

```sh
openssl req -x509 -newkey rsa:2048 -nodes -days 1 \
  -subj '/CN=juju-qa-extra-ca' -addext 'basicConstraints=critical,CA:TRUE' \
  -keyout "$QA_DIR/extra-ca.key" -out "$QA_DIR/extra-ca.crt"
jq -e --rawfile extra "$QA_DIR/extra-ca.crt" \
  --arg endpoint "$QA_API_ENDPOINT" '
  .[0] | select(.type == "k8s" and (."ca-credentials" | length) == 1) |
  {clouds: {"qa-k8s": (
    {type: "kubernetes", "auth-types": ."auth-types", endpoint: $endpoint,
     regions, config,
     "region-config": ."region-config", "skip-tls-verify": ."skip-tls-verify",
     "host-cloud-region": "lxd",
     "ca-certificates": (."ca-credentials" + [$extra])}
    | with_entries(select(.value != null))
  )}}
' "$QA_DIR/cloud-original.json" > "$QA_DIR/cloud.json"
juju update-k8s qa-k8s --client -f "$QA_DIR/cloud.json"
juju show-cloud qa-k8s --client --format json |
  jq -e --slurpfile cloud "$QA_DIR/cloud.json" \
    '(.[0]."ca-credentials" | sort) ==
     ($cloud[0].clouds."qa-k8s"."ca-certificates" | sort)'
juju add-cloud --controller qa-secret-controller qa-k8s
```

The client comparison must return `true`, and `add-cloud` must report that the
cloud was added, not that it already exists. Controller `show-cloud` does not
include CA certificates; the backend comparison below checks the certificates
retrieved from the controller.

Create a Kubernetes model and two secrets, updating one to make a second
revision. Inspect the built-in backend:

```sh
juju add-model qa-secrets qa-k8s
juju model-secret-backend auto
juju model-secret-backend
juju add-secret qa-first value=one
juju update-secret qa-first value=two
juju show-secret qa-first --revisions
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
Kubernetes backend for this model. The revision listing must show revisions
1 and 2. Both `jq` checks must return `true`: the count is two secrets, not
three revisions, and both certificates appear exactly once. The backend
commands must succeed without a query error.
Verify removal of the second user secret from Kubernetes. This exercises the
removal service's separate backend lookup and its use of the certificate
bundle. The listing comparison above directly checks that both certificates
are returned; a cleanup failure alone does not establish its cause.

```sh
sudo k8s kubectl -n qa-secrets get secret "${QA_SECOND##*:}-1"
juju remove-secret "$QA_SECOND"
sudo k8s kubectl -n qa-secrets get secret "${QA_SECOND##*:}-1" \
  --ignore-not-found -o name
juju debug-log -m qa-secrets --replay --no-tail
```

The first command must find the Kubernetes Secret. Removal is asynchronous:
repeat the `get secret --ignore-not-found` command until it produces no output.
Check the model log for secret backend initialization or content deletion
errors. Finally remove the model and disposable controller:

```sh
juju destroy-model qa-secrets --no-prompt
juju models
juju remove-k8s qa-k8s --client --controller qa-secret-controller
juju destroy-controller qa-secret-controller --destroy-all-models --no-prompt
```

Wait for model removal before checking `juju models` and removing the cloud.
Removing the cloud from both client and controller requests cleanup of the
credential's Kubernetes RBAC resources; check for cleanup warnings and verify
that those resources are gone. Check that the `qa-secrets` namespace has been
removed as well:

```sh
sudo k8s kubectl get namespace qa-secrets --ignore-not-found -o name
```

The additional CA was added only to Juju's cloud definition and client trust
bundle; it was not installed as a Kubernetes cluster CA. Kubernetes itself
remains installed and running. Remove the temporary files, including the test
credential:

```sh
rm "$QA_DIR/kubeconfig" "$QA_DIR/cloud-original.json" "$QA_DIR/extra-ca.key" \
  "$QA_DIR/extra-ca.crt" "$QA_DIR/cluster-ca.crt" "$QA_DIR/cloud.json" \
  "$QA_DIR/backends.json"
rm "$GOBIN/juju" "$GOBIN/jujuc" "$GOBIN/containeragent" \
  "$GOBIN/juju-metadata" "$GOBIN/jujuagentd" "$GOBIN/pebble"
rmdir "$GOBIN"
```

Remove `$QA_DIR` after verifying that it contains only disposable QA files.

## Documentation changes

None.

## Links

None.
