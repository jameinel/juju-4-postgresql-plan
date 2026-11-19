Thanks for the review.

1. Added a comment in 66aef8c256 explaining that closing the channel in
   `IsBroken` models a break between the initial check and the select.
2. Kept the mock setup explicit. The tests have different break setup
   between `Connection` and the cleanup expectations, and keeping the
   cleanup callback visible makes the synchronization barrier easy to see.
   The duplication is limited to these two tests.

The closed channel and cleanup barrier remain unchanged. The already-broken
case is covered separately, and the normal-break test checks the connected
state before and after the break. Its `Broken()` callback signals before it
returns, so the test does not guarantee that the select is already parked.

The controller-presence suite passed with `-race`; both break tests passed
100 race-enabled stress runs. Scoped lint and `txncheck` also passed.
