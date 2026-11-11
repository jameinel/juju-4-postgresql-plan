# PostgreSQL PoC: bootstrap after Charmhub API recovery

> Snapshot for the next session. `PLAN.md`, the stage files and
> `DECISIONS.md` remain authoritative if they differ from this handoff.

## Current state

- The code tree is clean on `pg-poc/09-bootstrap` at `f3340c6710`, based
  on upstream `bd970f693c`. The stack is local and has not been republished.
- At this tip, separate 142-package PostgreSQL and real-dqlite sweeps each
  had 140 passing packages, no failures and two packages without tests.
  Skips and excluded import/recovery packages remain; see
  [Stage 6](../stages/06-compatibility.md#catalog-teardown-dependency-check-2026-11-11).
- The PoC client and agent were rebuilt at this tip with
  `GOMAXPROCS=2 make juju jujuagentd`. The log is
  `/tmp/pg-poc-f334-bootstrap-build.log`.
- The prior failed controller and its two PostgreSQL schemas were removed
  with user approval. The latest inventory, before the deferred retry,
  showed no controller, model or test schemas and no other sessions.
  `public` and `login_hook` were not touched.
- The private bootstrap configuration is
  `~/.config/juju-pg-poc/bootstrap-75223c.json`, mode 0600. It contains the
  current database URL and must not be printed or copied into logs.
- On the last check, `charmhub.io` responded, but `api.charmhub.io`
  timed out from the host and an existing LXD controller. The user chose
  to wait rather than start another bootstrap while the API was unreachable.
  See [Stage 9](../stages/09-bootstrap.md#bootstrap-retry-preparation-2026-11-11-09f3340c6710).

## Resume

1. Recheck the code revision, both repository worktrees, private config
   permissions, controller names, LXD resources and the database inventory.
   The inventory-only command is:

   ```sh
   timeout 30s scripts/postgresql-poc/drop-juju-schemas.py
   ```

   Do not recreate the database or drop any schemas without inspecting
   the exact targets and obtaining the user's approval.
2. Verify HTTPS access to
   `https://api.charmhub.io/v2/charms/info/juju-controller` from the host
   and, if available, an LXD instance on the bootstrap network. If the API
   still times out, stop and report that connectivity remains blocked.
3. If the revision or build has changed, rebuild with `make juju jujuagentd`
   and record the revision. Bootstrap with the explicit built client,
   private configuration and `--build-agent`, preserving failed resources:

   ```sh
   set -o pipefail
   timeout 1500s "$HOME/go/bin/juju" bootstrap localhost pgpoc-pg \
     --config "$HOME/.config/juju-pg-poc/bootstrap-75223c.json" \
     --build-agent --keep-broken 2>&1 | tee /tmp/pg-poc-bootstrap-retry.log
   ```

   Do not use `--debug`, which can expose the database URL. Check progress
   at least every two minutes. If bootstrap fails or times out, inspect and
   preserve its resources; do not assume automatic cleanup is safe.
4. On success, check the controller API and controller unit status, and
   inventory PostgreSQL to confirm the controller and controller-model
   schemas exist. Record actual results, revision and redacted logs in
   [Stage 9](../stages/09-bootstrap.md); bootstrap completion alone does
   not establish the functional milestone.
5. Continue the agreed D54 workload and same-revision dqlite comparison
   only within the user's requested scope. The cross-model secret scenario,
   teardown and backend comparison remain outstanding.

Never print the URL, pass credentials as process arguments, push branches,
or publish PRs or comments without the applicable user approval.
