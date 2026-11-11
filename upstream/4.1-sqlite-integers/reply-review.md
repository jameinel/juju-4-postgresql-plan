# Unpublished review follow-up for juju/juju#23680

Destination: juju/juju#23680, pull-request conversation comment.
Action: create one new comment; do not resolve review threads.
Status: local draft only. Show the exact content and obtain explicit approval
before publishing. The code changes are committed but not yet pushed.
Review/validation date: 2026-10-29.
Follow-up commit: `0e9be8ddd7307662b369b077da04e332b9ff6ee9`.
Tested worktree: `../4.1-sqlite-integers`, follow-up on
`bd5379086660bf2f6095ae4dcfbc968c4e166c80`, upstream base
`9ea82c03bda2bbd4b99bfcf2c136dbabf6d9379c`.

---

Thanks for the review.

- Warning 1: extended `TestTextIDsParsedToIntegers` to populate every field
  and compare complete output rows for each affected table, including NULL
  cases. The existing relation-status tests already compare complete rows.
- Warning 2: the proposed 4.0-to-4.1 controller upgrade path is not
  supported. `validateControllerCanBeUpgradedTo` rejects changes to either
  the major or minor version. A separate 4.1 controller is bootstrapped
  with a fresh controller database; only the models migrate, with exported
  data transformed and imported into new model databases using the
  destination schema. The source controller database is neither reused
  nor upgraded. The developer-facing schema package documentation
  (`domain/schema/doc.go`) and PR description clarify this distinction;
  user-facing upgrade documentation is unchanged. These base-schema
  changes still do not provide an in-place conversion for existing 4.1
  beta databases.
- Suggestion 1: `parseID` classifies parsing errors as
  `coreerrors.NotValid`; `parseOptionalID` propagates that classification.
  Tests cover malformed required and nullable IDs and verify
  classification survives generated-transform wrapping.
- Suggestion 2: the revision-updater view test explicitly compares its row
  count with the application count, alongside the existing per-application
  count assertions.

Review follow-up validation:

```sh
go test -p 2 -race ./domain/modelimport/transformer/transforms/to_v4_1_0 ./domain/schema -run 'TestDeltasSuite|Test.*SchemaSuite' -count=1
go test -p 2 -race ./domain/modelimport ./domain/modelimport/state/model -run 'TestImportSuite/TestImportTransformsLegacyTextIDs|TestRoundTripSuite/TestImportExportRoundTrip' -count=1
make pre-check
golangci-lint run -j 2 --timeout 5m --new-from-rev=HEAD ./domain/modelimport/transformer/transforms/to_v4_1_0 ./domain/schema
```

All checks passed. `make pre-check` skipped static analysis; the explicit
targeted lint reported zero issues. The race-enabled revision-updater view
test completed 52 bounded stress iterations without failures. The schema
package documentation was checked with `go doc ./domain/schema`.
Race-enabled tests and targeted lint were rerun successfully before commit.

No user-facing documentation is changed. Live bootstrap/migration QA has
not been run for this update.
