Merge 4.1 into main.

### Included PRs

- #23728: preserve agent-binary registrations on conflict.
- #23740: write backup assertions and validate recovery archives.
- #23763: match resource names when looking up resource types.

### Conflicts resolved

None; the merge was conflict-free.

## Checklist

- [ ] ~Code style: no merge-specific code changes~
- [ ] ~Comments: no merge-specific design decisions~
- [ ] ~Go unit tests: no merge-specific code changes~
- [ ] ~Integration tests: no merge-specific code changes~
- [ ] ~doc.go added or updated in changed packages~

## QA steps

~Additional merge-specific functional QA.~ The merge has no conflict
resolutions or other merge-specific code changes.

## Documentation changes

The backup command documents assertion creation, and the new `juju recover`
command documents archive validation and the recovery provisioning limitation.

## Links

- [Agent-binary registration](https://github.com/juju/juju/pull/23728)
- [Backup assertion and recovery preflight](https://github.com/juju/juju/pull/23740)
- [Resource type lookup](https://github.com/juju/juju/pull/23763)

**Issue:** ~Not linked.~

**Jira card:** ~Not linked.~
