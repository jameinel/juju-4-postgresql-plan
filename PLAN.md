# Juju 4 on PostgreSQL: proof-of-concept plan

Status: **DRAFT, awaiting review**.

This file holds only what every session needs. Detail for one stage lives
in that stage's file under [stages/](stages/).

## Goal and scope

Bootstrap a Juju 4 controller, built from the code tree, whose controller
and model databases live in an external PostgreSQL. Controller config
selects the backend:

```sh
$(go env GOPATH)/bin/juju bootstrap lxd pg-poc --build-agent \
  --config db-backend=postgresql \
  --config db-url="$JUJU_PG_POC_DB_URL"
```

In scope (D1): LXD machine controllers including `enable-ha`, fresh
bootstrap only, and an external PostgreSQL reached via a URL. dqlite
remains the default and keeps working.

Out of scope: Kubernetes controllers, migrating a dqlite controller,
`dbrepl` / `dbreplaccessor`, credential rotation, the controller-charm
relation, production hardening and performance work.

## Documents

| Document | Contents | When to read |
|---|---|---|
| `PLAN.md` | Workflow, status and rules | Every session |
| [stages/](stages/) `NN-<slug>.md` | Goal, steps, status, restart notes and findings of one stage | The stage being run; findings of earlier stages it cites |
| [DECISIONS.md](DECISIONS.md) | Confirmed decisions (D*, N*) | Entries the stage cites; before proposing a design |
| [UPSTREAM-FIXES.md](UPSTREAM-FIXES.md) | dqlite defects the port exposed | When classifying a `bug` or `latent` change |
| [DIALECT.md](DIALECT.md) | SQLite and PostgreSQL differences (X*) and how each is handled | Before writing or fixing SQL; when a test fails only on PostgreSQL |
| [ANALYSIS.md](ANALYSIS.md) | dqlite coupling analysis and trade-offs | Sections the stage cites |
| [handoffs/](handoffs/) | Point-in-time handoff notes, superseded by the above | Only when asked |

## Repositories

- **Code**: `~/dev/go/src/github.com/juju/juju-4-postgresql`, remote
  `origin` = `jameinel/juju`. One working tree; switch branches in it. Do
  not add worktrees.
- **Plan**: `~/dev/go/src/github.com/juju/juju-4-postgresql-plan`, remote
  `origin` = `jameinel/juju-4-postgresql-plan`, branch `main`. Never commit
  plan documents to the code repository.

Sessions run in the code repository. The plan repository is a sibling
directory, so the session needs access to it (for example `/add-dir` in
the Copilot CLI).

## Starting a session

Kick-off prompt. Replace `N`, and name the PostgreSQL host controller only
for stages that need a live PostgreSQL (such as 3, 4, 6 and 8 to 10):

```
Read ~/dev/go/src/github.com/juju/juju-4-postgresql-plan/PLAN.md and run
Stage N following its session protocol. Use the <controller> controller.
```

## Session protocol

1. Read this plan, the stage file, the DECISIONS.md and ANALYSIS.md
   entries it cites, and the `AGENTS.md` files that apply to the code it
   touches.
2. Work only on the named stage, plus any fix to an earlier stage that it
   needs (see [stack workflow](#stack-workflow)). Do not start another
   stage, even if this one finishes early.
3. Read the stage's **Status**, **Restart notes** and **Findings**, and the
   findings of the earlier stages it cites.
4. Confirm the stage's open questions, and any new design choice, with the
   user before writing code. Add a decision to DECISIONS.md only after the
   user has confirmed it.
5. Commit code only on the stage's branch, or on the earlier stage a fix
   belongs to.
6. Classify every change made to get PostgreSQL working, whether in code,
   tests or schema:
   - `bug`: the dqlite behaviour is wrong today.
   - `latent`: dqlite is right today only because of SQLite leniency (type
     affinity, coercion, double-quoted strings) or a test that ignores an
     error.
   - `portability`: valid on dqlite and only invalid on PostgreSQL.

   Put each `bug` and `latent` change in its own commit, apart from
   PoC-only changes, so it can be proposed upstream on its own. Add it to
   UPSTREAM-FIXES.md, together with defects found but not fixed. Record
   `portability` changes in the stage's findings. Record every SQLite /
   PostgreSQL difference behind a change, or found and not yet handled,
   in DIALECT.md, including driver workarounds.
7. Before ending the session:
   - Update the stage file's **Status**, **Restart notes** (what is done,
     what is next, gotchas) and **Findings**, and the status table below.
   - Commit and push the plan repository.
   - Push the code branches (see [stack workflow](#stack-workflow)).

## Status

| # | Stage | Branch | Status |
|---|---|---|---|
| - | Plan documents | plan repository `main` | in review |
| 00 | [Base fixes](stages/00-base-fixes.md) | `pg-poc/00-base-fixes` | done, in review |
| 01 | [PostgreSQL provisioning script](stages/01-provision.md) | `pg-poc/01-provision` | in review |
| 02 | [Move schema to `domain/schema/dqlite`](stages/02-schema-move.md) | `pg-poc/02-schema-move` | in review |
| 03 | [PG driver, placeholder rewriting, errors](stages/03-driver.md) | `pg-poc/03-driver` | in review |
| 04 | [PG schema generation](stages/04-schemagen.md) | `pg-poc/04-schemagen` | done, in review |
| 05 | [Query portability fixes](stages/05-queries.md) | `pg-poc/05-queries` | in progress (X37, X38 open) |
| 06 | [PG test harness](stages/06-test-harness.md) | `pg-poc/06-test-harness` | in progress (test SQL fixes) |
| 07 | [`db-backend` / `db-url` config plumbing](stages/07-config.md) | `pg-poc/07-config` | in review |
| 08 | [PG dbaccessor worker and bootstrap](stages/08-dbaccessor.md) | `pg-poc/08-dbaccessor` | in progress (bootstrap layer committed) |
| 09 | [LXD bootstrap and smoke test](stages/09-bootstrap.md) | `pg-poc/09-bootstrap` | not started |
| 10 | [enable-ha](stages/10-ha.md) | `pg-poc/10-ha` | not started |
| 11 | [Future work](stages/11-future-work.md) | n/a | not planned |

Next: resume from
[handoffs/2026-10-25-stage-6-remaining.md](handoffs/2026-10-25-stage-6-remaining.md):
the open Stage 5 fixes (X37, X38), then the Stage 6 test SQL commits and
a full rerun one package at a time, then continue Stage 8.

## Stack workflow

Every stage branch sits on the one below it in the table, in one line
(N4, N8). The base is upstream `4.1` at `a8591fe1b8`. The **top** is the
highest stage branch that exists, currently `pg-poc/08-dbaccessor`.

- **New stage**: create its branch from the top
  (`git switch -c pg-poc/NN-<slug> <top>`).
- **Fix that belongs to an earlier stage**:
  1. `git switch pg-poc/NN-<slug>` and commit the fix there.
  2. `git switch <top> && git rebase --update-refs pg-poc/NN-<slug>`. This
     replays the later stages on the fix and moves every branch between.
- **Amend an existing commit**: from the top, commit with
  `git commit --fixup=<sha>`, then run
  `GIT_SEQUENCE_EDITOR=true git rebase -i --autosquash --update-refs a8591fe1b8`.
- **New upstream base** (only when the user asks): run
  `git rebase --update-refs --onto <new-base> a8591fe1b8 <top>`, then
  update the base commit here.
- After any restack, run `go build ./...` on the top and the tests of the
  packages the fix touched.
- **Push** every stage branch together:

  ```sh
  git push --atomic --force-with-lease origin \
    $(git for-each-ref --format='%(refname:short)' 'refs/heads/pg-poc/*')
  ```

- Commit or stash before switching branches. GoLand re-indexes only the
  files that change, so switching between adjacent stages is cheap.
- To read another stage without switching, use
  `git show pg-poc/NN-<slug>:<path>` or `git diff pg-poc/NN-<slug> -- <path>`.
- `pg-poc/00-base-fixes` holds only fixes for pre-existing problems that
  are unrelated to the PoC. Stage 2 stays purely mechanical so it can be
  reviewed, and possibly proposed upstream, on its own.

## Environment

- `juju` on `PATH` is Juju 3.6. `juju_4` (snap) is a Juju 4 reference, for
  comparing behaviour with dqlite only.
- Build PoC binaries from the code tree with `make juju jujuagentd`.
  Always invoke the client by explicit path, `$(go env GOPATH)/bin/juju`,
  to avoid collisions with the snaps. `bootstrap --build-agent` runs
  `make jujuagentd` in the tree.
- The Juju 3.6 controller hosting PostgreSQL is a parameter (for example
  `PG_HOST_CONTROLLER`). Never hardcode controller names in scripts.
  Stage 1 describes the deployed models.
- Juju 3.6 and Juju 4 clients may share `~/.local/share/juju`; take care
  when switching controllers.
- Read third-party source (charms, libraries) from local clones under
  `~/dev/go/src/github.com/<owner>/<repo>`, beside the code repository,
  rather than through the GitHub API. Clone there if missing. Already
  cloned: `canonical/postgresql-operator` (16 charm on branch `16/edge`),
  `canonical/postgresql-single-kernel-library` (the charm's logic) and
  `canonical/data-integrator`. Go module sources are in the module cache.
- `psql` is installed on the host.
- Agent sessions see `~/.local/bin` on `PATH`, but not `~/go/bin`. The lint
  tools for `STATIC_ANALYSIS=1 make pre-check` (golangci-lint 2.14.0,
  `gci`, `govulncheck`, `go-licenses`) are in `~/.local/bin`; `shellcheck`
  is in `/usr/bin`. The `go` snap provides Go 1.27; use its `gofmt`, not an
  older one.
- gci:
  `gci write --section standard --section default --section "Prefix(github.com/juju/juju)" <files>`.

## Secrets

- Never commit a DB URL or password, in either repository.
- Keep the URL in an environment variable (`JUJU_PG_POC_DB_URL`) or a file
  outside the repositories with mode 0600 (the Stage 1 script writes
  `~/.config/juju-pg-poc/db-url`).
- Scripts print credentials only to stdout or a file the user names.

## Testing on PostgreSQL

- Opt-in PostgreSQL tests read the URL from `JUJU_POSTGRESQL_TEST_URL`
  (N6): `JUJU_POSTGRESQL_TEST_URL=$(cat ~/.config/juju-pg-poc/db-url)`.
  With it set, every suite built on `DqliteSuite` runs on PostgreSQL, not
  only the verified ones.
- Prove a run reached PostgreSQL: point the variable at
  `postgres://nobody@127.0.0.1:1/none?connect_timeout=2` and expect
  "connection refused". A Stage 6 run once silently used SQLite.
- Run one package at a time, with
  `timeout 200 go test -count=1 -timeout 3m -v ./pkg/`. A `SetUpTest`
  failure repeats for every test.
- The snap `go` writes nothing when stdout is redirected to a file. Pipe
  instead: `go test ... 2>&1 | cat > log`.
- PostgreSQL aborts the whole transaction on any statement error. Resolve
  conflicts in SQL (`ON CONFLICT`) rather than catching the error (X18).
- pgx will not bind Go integers to TEXT parameters (X23).
- Other SQL that only PostgreSQL rejects is listed in DIALECT.md.
- Domain errors mask driver errors, so tests match message text that both
  backends share.
- After changing dqlite DDL or overrides, regenerate with
  `go generate ./domain/schema/...` (triggers, then PostgreSQL DDL), or
  `go generate ./domain/schema/postgresql.go` for the PostgreSQL DDL only.
  The `pgschemagen` tests fail if the generated files are stale.
