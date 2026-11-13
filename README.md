# Juju 4 on PostgreSQL: proof-of-concept plan

Planning documents for running Juju 4 controllers on an external
PostgreSQL instead of dqlite. The code lives on the stacked
`pg-poc/NN-<slug>` branches of
[jameinel/juju](https://github.com/jameinel/juju).

Start with [AGENTS.md](AGENTS.md) for shared ownership and Git rules.
This is one planning project: the PoC and upstream workflows share direction
but retain separate acceptance and operating instructions.

- PoC implementation: start with [PLAN.md](PLAN.md).
- Stock upstream 4.1 fixes: start with [UPSTREAM-PLAN.md](UPSTREAM-PLAN.md)
  and its linked `upstream/<branch>/status.md`, plus the target repository's
  instructions; no PoC setup or QA is required. Branch directories also hold
  `pr.md` and `reply-<topic>.md` when needed. Workers maintain those records;
  the coordinator maintains the shared summaries.
  Correct the SQL/schema root cause, not just its Go conversion boundary.
  These fixes remain SQLite/dqlite-only; PostgreSQL is an optional diagnostic
  reference, not an upstream dependency.
- Implemented upstream work, decisions and rationale: [UPSTREAM-DONE.md](UPSTREAM-DONE.md).
- Evolving findings, provenance and landing status: [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md).
- Bringing landed changes into the PoC: [UPSTREAM-INTEGRATION.md](UPSTREAM-INTEGRATION.md).
- General Juju 4.1 follow-ups, outside the PoC and breakout plans:
  [JUJU-4.1-FOLLOWUPS.md](JUJU-4.1-FOLLOWUPS.md).
