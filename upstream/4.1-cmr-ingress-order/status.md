# 4.1-cmr-ingress-order

[Shared direction](../../AGENTS.md) | [Upstream dashboard](../../UPSTREAM-PLAN.md) | [Validation](../../.github/skills/upstream-breakout/references/validation.md)

## Current state and next actions

Observed 2026-11-20. The user authorised this session to take over and
extend the patch to the egress getter; earlier ingress-only scope is
superseded for this branch.

- Owner/session: `a738cfe3-142f-4810-a9ec-22d6556fb5ec`.
- Branch: `4.1-cmr-ingress-order`.
- Worktree: `.worktrees/4.1-cmr-ingress-order` under the primary checkout.
- Base: `bd970f693c8e5060973016fe9059108570fa5bd3`.
- Local head: `62a12cdc6a82db6539c4100a3aee09bb234f326f`.
- Remote branch head: `62a12cdc6a82db6539c4100a3aee09bb234f326f`.
- Phase: follow-up, PR title/description and review reply published.

Jenkins build 40941 ran PR 23805 at local head
`62a12cdc6a82db6539c4100a3aee09bb234f326f`. The only reported test failure
was `TestWatcherSuite/TestWatchRelationEgressNetworks`, timing out while
waiting for the changestream to become idle after setup step 5. The failure
is in the watcher test harness and not in the CIDR ordering query or contract.
A local `-race -count=20` run of that test passed. It is therefore recorded
as separate watcher-test flake follow-up in `UPSTREAM-PLAN.md`, not as a fix
to this PR. No code change was made on this branch. The PR should be rerun
before merge if Jenkins remains the only unmet CI gate.

[#23805](https://github.com/juju/juju/pull/23805): Simon Richardson
approved the published head without explicitly answering the historical
ordering-contract question. Van Cleef made three suggestions; the local
follow-up orders egress getter results, completes the relation-life comment,
and renames the ingress ordering test. The egress watcher initial query and
fallbacks are not covered by the getter's ordering contract.

The user requested a commit-message-only amendment on 2026-11-16.
Local head `62a12cdc6a82db6539c4100a3aee09bb234f326f` has the same tree
(`066cef54c14e91220d5fcb5b13b4489e5cc840bf`) as the tested
`55aba2d24873cac419e232c7eed2dc6603255081`:

- `go test -race -count=1 -timeout 5m ./domain/crossmodelrelation/state/model ./domain/crossmodelrelation/service` passed on SQLite.
- `make run-go-tests TEST_PACKAGES='./domain/crossmodelrelation/state/model ./domain/crossmodelrelation/service' TEST_ARGS='-race -count=1 -timeout 8m' TEST_FILTER='.'` passed on real dqlite with `dqlite,libsqlite3` tags.
- `golangci-lint run -c .golangci.yml ./domain/crossmodelrelation/state/model/... ./domain/crossmodelrelation/service/...`, `txncheck "$PWD"`, gci, and `git diff --check` passed.
- `make pre-check` ran but skipped full static analysis. PostgreSQL was not
  run. The user reports that two-controller manual QA passed.

The published head has 12 failed checks, including static analysis, and a
pending Jenkins status; failure causes remain uninvestigated. Unit Tests
passed. The new head has not yet been assessed in CI. The approved PR
title and description were published, and the approved bot reply was
[posted](https://github.com/juju/juju/pull/23805#issuecomment-6926891862).
Triage CI and seek maintainer clarification on the original contract.
See `pr.md` and `reply-van-cleef.md`.

### Manual QA observation, 2026-11-16

The user ran the documented setup and SQL, but both expected grep commands
returned no lines. Read-only inspection of `x17-qa` showed two egress rows
(`10.10.0.0/16`, `10.2.0.0/16`) in the consuming model and no ingress rows
in the offering model. The offered `dummy-source:sink` endpoint is a requirer;
the local `dummy-sink:source` is a provider. The offering firewaller logs
that it skips a local requirer, and the consuming worker repeatedly fails to
find the external controller for the offering model. This setup therefore
does not exercise the two log paths in the draft.

The installed controller agent is `4.1-beta3.1`. An initial check mistakenly
examined `jujud`, which is a compatibility shim, rather than the running
`jujuagentd`. The deployed `jujuagentd` matches the SHA-256 of
`/tmp/tmp.ZeVVyPCf1D/jujuagentd`, contains both branch SQL ordering markers,
and has Go build revision `55aba2d24873cac419e232c7eed2dc6603255081`.
The run used the local follow-up; it did not exercise the ordering paths
because of the endpoint roles. The controller lookup failure remains to be
assessed. No controller or model was modified during this investigation.

The same-controller lookup failure is explained by the current code:
`application.saveExternalController` deliberately does not save this
controller as external, but the firewaller adapter's
`ControllerAPIInfoForModel` only calls the external-controller lookup.
The observed provider-side worker therefore cannot obtain local-controller
connection details. This is separate from CIDR ordering and needs separate
triage; a two-controller retry may avoid that particular lookup path but
is not a fix for it. At the user's direction, `pr.md` uses two controllers
with the provider offered and the requirer consuming. The user reports that
the manual QA passed. Both controller `jujuagentd` binaries have the same
SHA-256 as `/tmp/tmp.ZeVVyPCf1D/jujuagentd`, whose build revision is
`55aba2d24873cac419e232c7eed2dc6603255081`.

## Recorded scope and evidence

The following entries preserve source snapshots, including superseded
heads. Use the current-state section above for the latest recorded action.

<!-- Migrated from UPSTREAM-PLAN.md; historical evidence retains its revision. -->

### `4.1-cmr-ingress-order` — X17, explicit CIDR ordering

**Status:** guaranteed lexical ordering chosen by the user on 2026-11-10
and implemented locally in the PoC and upstream
[#23805](https://github.com/juju/juju/pull/23805). Upstream intent remains
unconfirmed; review, CI and functional QA are pending.

`GetRelationNetworkIngress` in
`domain/crossmodelrelation/state/model/relation_network.go` selects CIDRs
without ordering, while `TestGetRelationNetworkIngressSortedOrder` explicitly
expects sorted output. The primary key is `(relation_uuid, cidr)`, which can
yield incidental index order on SQLite; the firewaller consumes a set.

The local decision is to preserve the sorted behavior explicitly with
`ORDER BY cidr`, document lexical ordering in the state/service contracts,
and retain coverage distinguishing lexical from numeric address ordering.
Do not extend this decision to sibling egress queries.

**Required upstream review question:** was sorted output strictly intended
as an API contract, or was the test capturing incidental SQLite ordering?
When preparing the upstream fix, explain the local decision and ask
maintainers to confirm the intended contract. Record their response and
report it back to the user; do not present the historical intent as settled.
If upstream intends an unordered collection instead, revisit the local
decision with the user before changing direction.
