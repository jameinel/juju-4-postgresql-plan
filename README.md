# Juju 4 on PostgreSQL: proof-of-concept plan

Planning documents for running Juju 4 controllers on an external
PostgreSQL instead of dqlite. The code lives on the stacked
`pg-poc/NN-<slug>` branches of
[jameinel/juju](https://github.com/jameinel/juju).

- PoC implementation: start with [PLAN.md](PLAN.md).
- Stock upstream 4.1 fixes: start with [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md)
  and the target repository's instructions; no PoC setup or QA is required.
- Evolving findings, provenance and landing status: [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md).
- Bringing landed changes into the PoC: [UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md).
