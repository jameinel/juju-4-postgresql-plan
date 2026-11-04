> Unpublished local draft recovered on 2026-11-04. The refactor remains
> uncommitted; validation statements below are historical session claims,
> not new checks. Final lint, compilation and stress were recorded as pending.
> Do not publish without reviewing the current code, results and exact text.

Destination: general comment on https://github.com/juju/juju/pull/23731
Action: proposed review response only; NOT published.

On suggestion 1: I consolidated the database work into a private scalar
allocator shared by NextValue and NextNValues. NextValue returns its value
directly; NextNValues constructs the requested range, so single-value
callers do not need to go through the batch abstraction.

The shared allocator uses INSERT ... ON CONFLICT ... RETURNING to obtain
the last allocated value without a second SELECT. It still handles both
first use and subsequent increments in the caller's transaction. Reading
a returned row confirms that allocation occurred; a regression checks that
suppressed inserts and updates fail for both entry points without changing
the persisted sequence. Zero-count batches remain a no-op.

ON CONFLICT is not the only portable implementation, but UPDATE alone
cannot initialise a namespace, and native database sequences are not a
drop-in replacement for these transactional rollback guarantees.

On suggestion 2: the original test could not pass after a transaction error,
because it asserts eg.Wait() before examining results. Still, I changed it
to send results only after the entire transaction succeeds, rather than
merely checking NextValue's error inside the transaction.

On suggestion 3: I added concise suite comments distinguishing allocation
and persistence regressions from PostgreSQL SQL acceptance/execution
checks. The PR already explicitly states that PostgreSQL was not exercised.
I also added a comment explaining why allocation uses an upsert: it handles
first use and subsequent increments in the caller's transaction.

Validation: the refactor passes all sequence package race suites, including
the new missing-returned-row regression. Updated lint, compilation and
bounded stress results are pending; the previous review adjustment passed
36 concurrent-test stress runs with zero failures.
