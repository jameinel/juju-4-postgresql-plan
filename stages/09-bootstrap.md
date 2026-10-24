# Stage 09: LXD bootstrap and smoke test

Branch `pg-poc/09-bootstrap`, to be created on `pg-poc/08-dbaccessor`.

- Goal: end-to-end proof.
- Steps:
  1. `make juju jujuagentd`.
  2. Bootstrap as shown under
     [Goal and scope](../PLAN.md#goal-and-scope).
  3. Smoke test: `add-model`, deploy a simple charm, `status`,
     `remove-application`, `destroy-model` (schema dropped).
  4. Run the same steps with `juju_4` on dqlite and compare.
- Verification: the smoke test passes; findings are recorded here.
- Status: not started.
- Restart notes: none yet.
