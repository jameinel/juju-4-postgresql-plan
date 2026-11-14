# Handoff: sqlair placeholder proposals (2026-11-14)

This is a historical handoff. The status records below have the current
state. The active SQLair worktrees are `numbered-placeholders`,
`placeholder-style-option` and `agent-guidance`; superseded worktrees were
removed while their branches were retained.

From session `7bbea3b5-54ae-4766-8d29-6c2f341eda69`. Records:
[dollar status](status.md), [style status](../sqlair-placeholder-style/status.md),
[agent guidance](../sqlair-agent-guidance/status.md).

## Current state

Repository: `~/dev/go/src/github.com/canonical/sqlair`; base `main` `bd54c49`.
All branches are local and unpushed; each worktree is clean.

| Branch | Worktree | Head | vs `main` |
|---|---|---|---|
| `numbered-placeholders` | `.worktrees/numbered-placeholders` | `2760dcf` | 4 commits, +508/-146 |
| `placeholder-style-option` | `.worktrees/placeholder-style-option` | `1fe3704` | shares first 3, +1 commit; +674/-108 |
| `agent-guidance` | `.worktrees/agent-guidance` | `bc156ff` | +70 (`AGENTS.md`) |

Shared commits: `ead635c` (numbering), `50117a0` (reject raw `$N`),
`eb8381a` (`tests/postgresql` module, CI, CONTRIBUTING).

Planning commit `71cc0aa` (local, unpushed) holds the drafts and records.
`upstream/sqlair-placeholder-style/pr.md` has an uncommitted user edit (the
sentence about the abandoned `?` implementation); keep it and commit it with
the next planning change. Other sessions' dirty files (`PLAN.md`, some
`UPSTREAM-PLAN.md` hunks, `upstream/4.1-postgresql-tests/*`) are not ours;
stage only named owned files.

## User decisions from the last turn

1. **Push approved, not yet run.** Push `numbered-placeholders` and
   `placeholder-style-option` to `jameinel/sqlair`. The only remote is
   `origin` (canonical/sqlair, SSH), so push by URL:

   ```sh
   git push git@github.com:jameinel/sqlair.git \
       numbered-placeholders placeholder-style-option
   ```

   Do not delete the old fork branches (`dollar-placeholders`,
   `placeholder-style`, `placeholder-styles`,
   `postgresql-test-infrastructure`). Do not push `agent-guidance`.
   If the branches change after this push, a force push needs new approval.
   Pushing should trigger fork CI, which exercises the `postgresql` job.
2. **`reply-issue-189.md` content is approved** for posting once the PR is
   open. Fill in `#<PR>` and confirm before posting.
3. **Style `pr.md` is approved** apart from the open QA setup item below.

## Open work

### Replace Docker QA setup with Juju

The user prefers a Juju-deployed `postgresql` charm with `data-integrator`
and has not vetted Docker. Docker is not installed here, so the Docker
commands were never run; QA ran against the user's existing URL
(`~/.config/juju-pg-poc/db-url`; never print it).

Docker appears in:

- `upstream/sqlair-dollar-placeholders/pr.md` setup (~lines 56-60) and
  cleanup (~line 170);
- `upstream/sqlair-placeholder-style/pr.md` setup (~62-66) and cleanup
  (~187). The user said they were otherwise happy with this file; confirm
  before changing it, though the same concern applies;
- `CONTRIBUTING.md` "Testing with PostgreSQL" (~lines 81-100), in shared
  commit `eb8381a`. Changing it rewrites both branches; ask whether to use
  Juju, generic "any PostgreSQL URL" wording, or keep Docker after vetting.

Unverified outline for the QA steps (check against the user's deployment
and current charm docs before use):

```sh
juju add-model sqlair-qa
juju deploy postgresql --channel 16/stable
juju deploy data-integrator --config database-name=sqlair_qa
juju integrate data-integrator postgresql
juju wait-for application data-integrator --query='status=="active"'
juju run data-integrator/leader get-credentials
```

Verify how to build `SQLAIR_POSTGRESQL_TEST_URL` from the credentials
output (`uris`, or username/password/endpoints), whether the relation user
can `CREATE SCHEMA` under the PostgreSQL 16 charm's role model (the tests
create a schema per test), whether TLS/`sslmode` matters, and the cleanup
(`juju destroy-model sqlair-qa --no-prompt --destroy-storage`).

### Decide the Go version

Facts:

- Root `go.mod` is `go 1.24` (upstream's existing value, unchanged).
- `tests/postgresql/go.mod` is `go 1.25.0`, set by `go mod tidy`; believed
  to be pgx's minimum (not yet confirmed from pgx's `go.mod`).
- CI test matrix is `[1.24.x, 1.x]`; the `postgresql` job uses `1.x`.
- CONTRIBUTING says the module "requires Go 1.25 or later"; both `pr.md`
  files say the root module "stays on Go 1.24" (~line 26 dollar, ~42 style).

The user notes Go 1.26 is supported upstream and is the Ubuntu LTS version,
with a longer support window. With Go 1.27 current, 1.24 and 1.25 are both
past upstream support. Earlier this session the bump was deferred to keep
the PR minimal; the old `postgresql-test-infrastructure` branch has a Go
1.26 commit for reference.

Options to put to the user:

- bump only `tests/postgresql` to 1.26 (no consumer impact);
- bump root and test modules plus the CI matrix (`[1.26.x, 1.x]`),
  CONTRIBUTING and both `pr.md` files, as a separate commit at the start of
  the shared prefix or as a separate PR. This raises the minimum Go for
  sqlair importers; check Juju's `go.mod` to confirm Juju is unaffected.

### Still open from earlier

- Real-dqlite validation (Juju build with a `replace` directive).
- Split the PR or accept the squash merge.
- README "Database agnostic" bullet vs the SQL Server regression.
- Old GitHub Action pins (`setup-go@v4`/`v5`).
- en_GB docs spellcheck vs US text; mixed license headers.
- `sqlair-empty-in` follow-up.
- Optional: propose `AGENTS.md`; delete old fork branches; push the
  planning commit. Each needs explicit approval.

## Validation commands

Run in each code worktree (pipe Go output through `tail`; Go 1.27.4 snap):

```sh
gofmt -l . && go vet ./... && go test ./...
cd tests/postgresql && go mod tidy -diff && go vet ./... && \
  SQLAIR_POSTGRESQL_TEST_URL="$(cat ~/.config/juju-pg-poc/db-url)" go test ./...
```

Docs spelling: `make spelling` in `docs/` flags only the existing
`security.md` words (Cryptographic, TLS, utilizing).
