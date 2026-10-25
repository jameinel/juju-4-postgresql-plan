# Comparative testing milestone

Status: design outline; workloads and acceptance thresholds not selected.
Read when planning or running Stage 9 comparisons. This is a PoC comparison
of Juju implementations and deployments, not an isolated database-engine
benchmark (D38).

## Cases

| Case | Build | Purpose |
|---|---|---|
| Upstream | Stock Juju at a pinned upstream revision | Baseline before or after upstream fixes land. |
| PoC dqlite | PoC source revision, default backend | Identify effects of shared portability changes versus upstream. |
| PoC PostgreSQL | Same PoC source revision, PostgreSQL selected | Compare backend implementations with shared Juju code controlled. |

Record all revisions, build options and agent binaries. Upstream fixes
will shrink the difference between stock and PoC dqlite; never silently
change the baseline within a comparison. A reference snap is useful only
with its exact revision/version and differing build conditions recorded.

## Order and correctness gate

Start with single-controller deployments; Juju controller HA follows in
Stage 10 (D39). Before calling the PoC functional:

- Stage 8 runtime integration and namespace lifecycle must work.
- Stage 9's end-to-end correctness scenario must pass, including normal
  model creation and completed teardown.
- The dedicated cross-model secrets fix and its multi-unit regression
  tests must be complete (D40), not hidden by avoiding that workload.
- Required failures must be resolved or explicitly scoped out by the
  user. PostgreSQL model import/export is deferred; do not include it in
  the initial workload. Upstream import/export compatibility still matters
  for shared fixes.

Passing unit tests alone does not establish a functioning controller.
Do not compare timings from runs that fail correctness checks.

## Experiment contract to agree before execution

Choose one bounded workload first. Record:

- Operation, data shape/scale, concurrency and observable completion.
- Controller and database CPU, memory, storage, topology and network
  latency. External PostgreSQL consumes resources outside the controller;
  include them rather than comparing controller usage alone.
- Charm/channel/revision and any other workload artifacts.
- PostgreSQL version/configuration, connection limits, isolation and TLS
  mode; the corresponding dqlite settings.
- Warm-up, repetitions, run order, reset/cleanup and timeout policy.
- Metrics and collection method: candidate metrics are elapsed time,
  throughput, latency distribution, errors/retries and resource use.

Keep functional acceptance separate from performance observations. Do not
invent a PostgreSQL speedup target or optimise implementations to meet one.
Run cases without competing test loads. Report material deployment
asymmetries and uncertainty rather than attributing all differences to the
engine.

## Open questions

- Which workload best represents the first useful comparison: controller
  lifecycle, model/application scale, or a targeted state/query workload?
- What scale, repetitions and instrumentation are affordable on this host?
- Is TLS overhead a separate comparison after the plain baseline? D8 keeps
  both provisioning modes available but does not require mixing them in
  one result.

These questions do not block documenting fixes or completing runtime
integration. They must be settled before dispatching a benchmark task.
