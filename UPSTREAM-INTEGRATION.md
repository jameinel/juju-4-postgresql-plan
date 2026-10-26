# Bringing landed upstream changes into the PoC

Reusable procedure for the PoC integration agent. Read this file when
reconciling a landed batch, together with PLAN.md, OPERATIONS.md and only
the relevant entries in [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md).
That register owns all per-fix provenance, status and results; do not add
a second fix inventory here.

Upstream implementation agents use UPSTREAM-PLAN.md and their target
repository's guidance instead. This procedure is not part of their QA or
review acceptance.

## Inputs and timing

The coordinator supplies the selected landed fixes, verified upstream
merge commits, old PoC base/branch tips and proposed new upstream base.
Record those per-batch inputs in the register/session handoff, not as
permanent instructions in this file.

Prefer one reconciliation per coherent landed batch, not per leaf commit.
Unmerged PRs are not the upstream baseline; testing a combined integration
branch before landing needs a separate scope decision. Under D43, a PR may
already be cherry-picked unchanged into the PoC. When it lands, compare it
with the landed result, drop the duplicate commits and keep any residual
work.

## Procedure

1. **Prepare a recovery point.** Coordinate with active owners, pause PoC
   edits/restacks and check the working tree. Preserve the old stage tips
   and outstanding work before changing history. Do not move another
   owner's branch or checkout.
2. **Verify the incoming batch.** Refresh and pin upstream 4.1. Check that
   the selected fixes really landed and compare their final semantics and
   tests with the PoC versions. A matching subject or patch ID is not
   enough after a split, squash or review-driven change.
3. **Map what remains.** For each registered fix, distinguish equivalent
   shared changes, upstream differences needing reconciliation, and
   PostgreSQL-only residual work. Include helper and test dependencies;
   do not drop a whole mixed commit just because one fix landed.
4. **Reconcile the stack.** Use the current stack workflow in PLAN.md.
   Remove duplicate shared hunks while preserving needed residuals and
   stage references. Investigate conflicts, skipped commits and duplicated
   changes instead of accepting an automatic result blindly.
5. **Reconcile generated artifacts.** Account for schema-layout changes,
   regenerate PostgreSQL output/overrides from the resulting shared schema,
   and check export/import/recovery representations. Remove an override
   only when its purpose has disappeared. Do not choose one side's entire
   generated file as a shortcut through conflicts.
6. **Validate the result.** Verify adjacent stage ancestry, build the top,
   and run affected default-backend/PostgreSQL checks plus the applicable
   milestone gate from PLAN.md, using OPERATIONS.md. Retain new upstream
   regressions. A clean rebase or upstream test pass alone does not prove
   the PostgreSQL path still works.
7. **Record and publish.** Update each selected register entry with the
   upstream revision, residual changes, reconciled PoC revision and test
   evidence. Update the plan's base and verification status; an unverified
   new base must be labelled as such. Push the coherent stack under the
   normal publication protocol, then release the coordination hold.

## Stop conditions

Ask before resolving an intentional semantic difference, changing scope,
removing coverage, altering shared ownership or recreating a database.
If validation fails, retain the recovery point and record the blocker;
do not claim the batch integrated successfully or hide it with a skip.
Integration failures return to their owning PoC task, not to an unrelated
upstream agent as a new backend-testing requirement.
