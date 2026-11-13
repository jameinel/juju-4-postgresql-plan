The data stored doesn't have a stream id. So if we wanted to make the code
correct for its namesake, we'd have to change the data structure. I've gone
with your suggestion to rename the function to make it clearer what we are
looking at. It was injecting the stream id to match the result shape returned
by the model lookup. The model lookup checks the model's target stream, while
cached controller binaries have no recorded stream. I think returning the
stored controller fields and processing them separately in the caller is
cleaner than injecting the stream in this function.

The controller store does not record stream membership: its rows and
registration input contain version, architecture and object UUID, but no
stream. The previous projection echoed the requested stream without filtering
the rows, so it could not establish that the binaries belonged to that stream.

I renamed the controller lookup to `GetAllAgentStoreBinaries`, removed the
stream parameter, and return only the stored version and architecture. Its
callers still use all controller-store binaries, preserving existing behavior.
The model lookup retains its stream parameter because it checks the requested
stream against the model's target stream in the same transaction as the binary
query.

This makes the controller lookup's contract explicit, but it does not make
the higher-level stream-specific upgrade checks filter controller-store
binaries: those checks can still select a stored version irrespective of
the requested stream. The model lookup also assumes its store applies to its
current target stream. Per-binary stream membership needs a separate design
for storage, existing rows and binaries shared across streams.

The affected state and service suites passed with `-race` on SQLite; the
state suite also passed with `-race` on real dqlite. Scoped lint and
transaction analysis passed. Live controller QA has not been run.
