Regarding the [review](https://github.com/juju/juju/pull/23884#pullrequestreview-5962536483):

The existing `date('now')` cutoff does include leases that expired earlier
the same UTC day, and the comment says they are removed even though the query
only filters them. This predates the fixture change.

I'd keep this PR test-only and address the cutoff and a same-day regression
separately. A `-time.Minute` fixture would cross UTC midnight for one minute
per day, so the regression should establish the boundary deterministically.
The production fix also needs to consider the database clock used for lease
writes and pinned-lease semantics before choosing a cutoff.
