# fix(schema): require keys for charm container mounts and manifest bases

Published after explicit approval on 2026-10-30:
https://github.com/juju/juju/pull/23710
Base: `4.1`; head: `jameinel:4.1-nullable-primary-keys` at `ba21f48ad3`.
The title above and body below match the approved published content;
these metadata paragraphs are local only. Further edits need approval.

Two charm tables have nullable columns inside composite primary keys.
SQLite permits these NULLs and does not enforce uniqueness over them;
standard SQL requires every primary-key column to be `NOT NULL`.

This PR contains only the charm-key corrections. The independent exposure
scope change and combined primary-key guard have moved to the stacked
`4.1-application-exposure-scope` PR. Land this PR first. The lookup-key
NOT NULL correction is a separate independent follow-up, not part of either
PR.

Rebased onto upstream `4.1` at `8369e6e2a9`, including the merged
integer-schema corrections. Charm head: `ba21f48ad3`.

### `fix(schema): require keys for charm container mounts and manifest bases`

- `charm_container_mount.charm_container_key` is `NOT NULL`. It is also the
  foreign key to `charm_container`, which a NULL bypassed.
- `charm_manifest_base` is keyed on `(charm_uuid, array_index,
  nested_array_index)`, the manifest position that the encoder already makes
  unique. The descriptive columns remain nullable attributes.
- The 4.0.12 to 4.1.0 import delta rejects a mount without a container key,
  reporting the mount index and charm UUID.
- The manifest-base delta rejects duplicate
  `(charm_uuid, array_index, nested_array_index)` positions with the charm
  UUID and both indices. The old key could distinguish these rows using
  descriptive attributes; selecting or dropping one would lose data.
- The transform generator explicitly routes this constraint-only change
  through its delta even though the exported struct shape is unchanged.
  Regeneration must not silently restore the old unchecked cast loop.
- Legacy decode/transform/import/export tests preserve nullable attributes
  and all populated fields. Duplicate identical or differing-attribute
  positions and NULL mount keys fail before import writes.
- Recovery round trips and charm schema tests cover the new constraints.

### Schema change policy

The 4.1 base schema is edited in place, as in juju/juju#23690 and
juju/juju#23682. 4.1 is in beta, the 4.1.0 payload is unreleased, and no
post-patch is registered. Controllers bootstrapped from an earlier 4.1 beta
cannot be upgraded in place. 4.0 models reach 4.1 by migration, which
applies the deltas.

### Out of scope

- `port_range.relation_uuid` also uses NULL as a wildcard, but it is not
  part of a primary key.
- A 4.0.12 payload with duplicate manifest positions or a NULL container
  key fails import. Juju does not write either.

## Checklist

- [x] Code style: imports ordered, good names, simple structure, etc
- [x] Comments saying why design decisions were made
- [x] Go unit tests, with comments saying what you're testing
- [ ] ~[Integration tests](https://github.com/juju/juju/tree/main/tests), with comments saying what you're testing~ (legacy import/export and recovery regressions included; live migration remains proposed)
- [ ] ~[doc.go](https://discourse.charmhub.io/t/readme-in-packages/451) added or updated in changed packages~

## QA steps

### Automated checks

```sh
make rebuild-export
make run-go-tests \
  TEST_PACKAGES="./generate/export ./domain/schema ./domain/modelimport/... ./domain/recovery/..." \
  TEST_ARGS="-race -count=1 -p=2 -timeout=8m" TEST_FILTER=Test
```

All nine affected race packages and explicit build-tagged focused lint
passed at `ba21f48ad3`. Regeneration was idempotent; focused charm-import
stress reported 132 runs and zero failures before its bounded timeout.
Live migration QA below has not been run.

Check that:

- Valid legacy mounts retain container key, storage and location after
  decode, transform, import and export; NULL keys name the charm and mount.
- Manifest bases preserve unset and populated descriptive attributes and
  distinct positions. Duplicate positions report the charm and both
  indices rather than a database UNIQUE error.
- Running `make rebuild-export` again preserves the manifest delta call.
- Schema tests reject NULL mount keys and duplicate manifest positions.
- Recovery dump/restore preserves mounts and bases.

### Live migration (proposed, not run)

Build the branch (`make install`) and bootstrap an LXD destination with
`juju bootstrap lxd dst41 --build-agent`. On a 4.0 controller deploy a
charm with manifest bases and container mounts backed by storage. Wait
for it to become active, then migrate its model to `dst41` using the 4.0
client. Confirm migration finishes, the application remains healthy,
`add-unit` succeeds and the storage mount is available on the new unit.
Destroy only the QA controllers created for this run afterwards.

## Documentation changes

No CLI/API workflow changes. Malformed legacy charm data fails migration
with a diagnostic rather than a database constraint error.

## Links

**Exposure-only comparison:**
https://github.com/jameinel/juju/compare/4.1-nullable-primary-keys...4.1-application-exposure-scope

~**Jira card:**~ None.
