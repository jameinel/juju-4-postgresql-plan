Published after explicit approval on 2026-10-30:
https://github.com/juju/juju/pull/23710#issuecomment-6527752522
Replying to https://github.com/juju/juju/pull/23710#pullrequestreview-5701485059.
The body below matches the approved publication. This metadata is local only.
No review threads were resolved; further published edits need approval.

Thanks for the suggestions. I've separated the two concerns:

- This PR now keeps the charm-key corrections and their delta/export/recovery
  tests. The exposure changes are on the stacked
  [`4.1-application-exposure-scope` branch](https://github.com/jameinel/juju/compare/4.1-nullable-primary-keys...4.1-application-exposure-scope),
  covering the four-table split and its producers, consumers, watchers,
  removal, import/export and recovery. The schema-wide primary-key guard is
  there because it only passes after both corrections.
- `CharmManifestBase` now uses a delta that rejects duplicate positions with
  the charm UUID and both indices, while preserving nullable attributes.
  Tests cover identical and differing-attribute duplicates and actual legacy
  decode-transform-import-export. The generator preserves this
  constraint-only delta after regeneration.
- `unsetAllExposedEndpoints` prepare/run errors now identify the SQL/table.
  Regression coverage injects a failure in each of the four deletion tables
  and verifies that earlier deletions roll back.
