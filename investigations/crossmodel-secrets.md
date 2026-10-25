# Cross-model secrets: U11 / X16 / X30

Status: dedicated work required before the functional milestone (D40).
No implementation is authorised by the candidate query shape below.
Owner: Stage 5 for the shared production fix and its regression tests;
use Stage 6's harness for dual-backend verification. No new branch line.

## Problem and entry points

`domain/crossmodelrelation/state/model/secrets.go` has two queries that:

- derive an application name with `substr(unit_name, 1,
  instr(unit_name, '/') - 1)` (SQLite-only X30);
- group by `sruc.secret_id` but select `sr.uuid` and test
  `sruc.current_revision` against `MAX(sr.revision)` (X16).

With several consuming units, SQLite can choose an arbitrary unit row.
This is a latent correctness issue, not just syntax portability. Read the
surrounding callers and tests before selecting the desired semantics.
The legacy offer-import test's use of `instr` was fixed independently;
it does not resolve these production queries.

## Exploration history

The 2026-10-25 test-SQL session proposed:

1. Match the application prefix with
   `substr(sruc.unit_name, 1, length(app.name) + 1) = app.name || '/'`.
2. Remove the grouping and use `sr.revision > sruc.current_revision`.
3. For query 2, select distinct secret IDs while preserving the optional
   `sr.uuid IN (...)` filter.
4. For query 1, restrict `sr.revision` to the maximum revision for that
   secret using a correlated subquery.

This would mean "any consuming unit is behind". It was deferred under
D35 because the behaviour change needs dedicated design and multi-unit
coverage. D40 brings it into the functional gate; it does not approve
that semantic choice. Historical source:
[handoff section 6](../handoffs/2026-10-25-stage-6-test-sql.md#6-deferred-crossmodelrelation-secrets-u11-latent).
The synthetic application name in those tests is its UUID, and consumer
unit names are `<uuid>/0`.

## Bounded tasks

**05-CMR-design:** trace both query outputs to their consumers. Establish
whether any/all consuming units should trigger an update, which revision
is returned, and how optional revision filtering interacts with that rule.
Present the intended behaviour and regression cases for user approval.
Stop here if those semantics cannot be established.

**05-CMR-fix:** after approval, implement only these queries and the
associated tests. Cover no consumers, one current/behind consumer, mixed
multi-unit revisions, multiple secrets/revisions, application-name prefix
collisions and the optional filter. Assert membership, revision selection
and absence of unwanted duplicates. Preserve supported input forms.
Verify the regression on SQLite/dqlite and the fixed behaviour on both
backends; follow repository watcher/race rules where the path needs them.

**Completion:** record the decision, commit, test evidence and U11 status.
A syntactically accepted query or passing single-unit test is insufficient.
Do not broaden this task into general cross-model relation redesign.

## Parallelism

Design/consumer analysis may run alongside unrelated runtime planning on
a pinned tree. Under N11 the upstream-only fix can be implemented in its
own worktree, after semantic approval; see [review branches](../UPSTREAM-PLAN.md).
It must not depend on the PG harness or edit the shared PoC tree. One
coordinator reconciles the landed result into the PoC; shared PostgreSQL
runs remain serial.
