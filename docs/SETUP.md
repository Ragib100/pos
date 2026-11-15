# Setup and Git guide

For everyone on the team, including people new to Git or Docker. Read this first, then `README.md` in this folder.

> **Planned commands.** The `make` commands below are created by task **B1-00** (BE1) in milestone M0. Until then they do not exist. This page describes the target setup so everyone works the same way from day one.

## A. One-time install (each PC)

Install these, then close and reopen the terminal.

| Tool | What it is | Install |
|---|---|---|
| Git | saves and shares code | Linux: `sudo apt install git` (or your distro's package) · Windows: git-scm.com |
| Docker | runs PostgreSQL (and later Redis) in containers | Linux: install Docker and Docker Compose, then add yourself to the `docker` group and log out and in · Windows: Docker Desktop with WSL2 enabled |
| uv | Python version and package manager | `curl -LsSf https://astral.sh/uv/install.sh \| sh` |
| Node LTS and pnpm | frontend tools | nodejs.org (LTS), then `npm i -g pnpm` |
| Claude Code | AI helper in the terminal | `npm i -g @anthropic-ai/claude-code` |
| Load-test tool (BE1 only, later) | load testing for M14 | k6 is a candidate; not decided |

Check: `git --version && docker --version && uv --version && pnpm --version` should print four versions. Windows users: run everything inside the **WSL Ubuntu** terminal, not PowerShell.

## B. Get the project (once)

```
git clone <repository-url>
cd <repository-folder>
git config pull.rebase true   # avoids "divergent branches" errors
make env                      # creates .env with random secrets (never commit it)
make up                       # starts PostgreSQL and the apps (first run is slow)
make migrate                  # applies database migrations
make seed                     # loads shop-type presets and a demo shop (from M1)
```

Planned services:

| Service | URL |
|---|---|
| Web app (Next.js) | http://localhost:3000 |
| API (Django) | http://localhost:8000/api/v1/ |
| API docs (OpenAPI) | http://localhost:8000/api/docs/ |
| PostgreSQL | localhost:5432 |

Secrets are generated for you in `.env`. Never commit `.env`; `.env.example` lists the variables and is committed.

## C. Every task: the loop

Work on a **short branch per task** and merge through a pull request (a change from the hackathon setup, where everyone worked on `main`; see `DECISIONS.md`, D-022).

```
git pull                                  # 1. get teammates' latest work
git switch -c b1-05-sale-transaction      # 2. branch named after the task ID
claude                                    # 3. open Claude Code, then type:  I am BE1. Do B1-05.
                                          #    Claude codes, tests and commits.
git push -u origin HEAD                   # 4. push and open a pull request
```

5. Another member reviews. Merge when CI is green. In the same pull request the task is ticked `[x]` in your file under `docs/tasks/` and a line is added to its Log.
6. Tell the group: "B1-05 merged". For the next task, type `/clear` in Claude Code (fresh context, fewer tokens), then go back to step 1.

**Reviews:** BE1 and BE2 review each other's migrations and RLS policies. FE1 and FE2 review each other's shared components. At least one review per pull request.

**Before you start a task**, check that its `Depends` milestones are done (all four tasks of each are ticked in `tasks/`) and that any gate it lists is answered (`PLAN.md`, section 3). Frontend tasks can start from the OpenAPI schema and a mock before the backend endpoints exist.

## D. What every pull request must pass

- Tests pass.
- Every shop-owned table has row-level security enabled and forced.
- No app imports another app's models directly.
- The committed OpenAPI schema is up to date (`make schema`).
- Lint passes, and the docs touched by the change are updated.

## E. Common fixes

| Problem | Fix |
|---|---|
| `permission denied ... docker.sock` | add yourself to the `docker` group, then log out and in |
| `port is already allocated` | `make down`, or stop whatever is using that port |
| push `rejected` / pull says `divergent branches` | someone pushed first: `git pull --rebase`, then push (run `git config pull.rebase true` once) |
| merge conflict | open Claude Code and say: `I have a merge conflict, help me fix it` |
| two migrations for one app conflict | tell Claude Code; it will merge them with a new migration. Never edit a migration that is already merged |
| CI says "table has no RLS" | the new table's migration is missing its policy: add it (see `ER-DIAGRAM.md` and `CLAUDE.md`) |
| CI says "schema out of date" | run `make schema` and commit the result |
| Claude says a task is blocked | its `Depends` milestone is not done yet. Pick your next unblocked task, or build against the mock |
| Want an older version back | use the release tag: `make deploy VERSION=<tag>` (added in M14) |
