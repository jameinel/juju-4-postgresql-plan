1. Replaced the adjacent `IFNULL` with `COALESCE` in this query. The other
   `IFNULL` occurrences remain outside this PR.
2. Updated the proposed PR description to explain that the secondary unit
   fixture now uses the subordinate application and charm.

The password query has also been simplified to select the instance ID
directly and use the absence of a matching row for invalid credentials,
rather than using `COUNT` and `MAX`. The affected SQLite and dqlite package
suites pass.
