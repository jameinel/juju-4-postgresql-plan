Re: [Van Cleef's review](https://github.com/juju/juju/pull/23807#pullrequestreview-5870181396).

1. Both ensure methods now return the normal wrapped query error for
   `ErrNoRows`, rather than treating it specially. Machine and unit removal
   do have key-row delete paths, although that does not by itself establish
   that a delete can make this read miss within the transaction.
2. Added comments explaining that the conflict-safe insert keeps the
   transaction usable and the subsequent read returns the winner. The
   aborted-transaction concern applies to PostgreSQL; SQLite's usual ABORT
   conflict handling rolls back the statement rather than the transaction.
3. I kept the entity-specific paths separate. A shared transaction helper
   would need the three prepared statements, the entity name and error,
   and a callback for the different Sqlair record types, for example:

   ```go
   ensureVirtualHostKey(ctx, db, machineName, "machine",
       machineerrors.MachineNotFound,
       getMachineUUIDStmt, insertStmt, getKeyStmt,
       func(uuid string) any {
           return machineVirtualSSHHostKey{
               MachineUUID: uuid,
               AlgorithmTypeID: algorithmTypeID,
               SSHKey: sshKey,
           }
       })
   ```

   The helper would also leave the entity-specific SQL preparation in each
   caller. The indirection and nine arguments make it less clear than the
   existing duplication.

I also kept the unconditional read rather than the suggested `RETURNING`
alternative: it has a single read-back path for both insert and conflict.
The SQLite and real-dqlite state suites passed with `-race`; each backend
also passed 10 stress runs of the two concurrent host-key tests.
Scoped lint, txncheck and the enabled pre-check passed, although the pre-check
skipped licence analysis, shfmt and govulncheck because those tools were
unavailable. A subsequent direct licence check passed, with warnings that
non-Go code could not be inspected for further dependencies. A direct shfmt
3.12.0 check reported 30 existing shell files; CI uses 3.7.0. The published-head
CI vulnerability check reported GO-2026-6859; I am leaving that check to CI.
