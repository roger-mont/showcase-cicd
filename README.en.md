# Delivery Platform — standard CI/CD and Developer Experience for a law firm's apps

![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-2088FF?logo=githubactions&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?logo=docker&logoColor=white)
![GHCR](https://img.shields.io/badge/GHCR-181717?logo=github&logoColor=white)
![Bash](https://img.shields.io/badge/Bash-4EAA25?logo=gnubash&logoColor=white)
![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?logo=postgresql&logoColor=white)
![Claude Code](https://img.shields.io/badge/Claude_Code-D97757?logo=anthropic&logoColor=white)
![MCP](https://img.shields.io/badge/MCP-server-555555)
![Private code · showcase](https://img.shields.io/badge/Private_code-showcase-6e7781)

🇧🇷 [Versão em português](README.md)

## Context

A law firm in Manaus, Brazil, runs about a dozen in-house applications (Django, Next.js, Node) on a Docker server behind Cloudflare Tunnel. Every app shipped its own way: manual deploys on the server, compose files written in different styles, no shared check before reaching production and no standard path to roll back. This platform brings two pieces together: a **central pipeline repository** designed to be reused by every app (CI, image publishing and deploys with automatic rollback), already running in production for the pilot app with an adoption playbook for the others; and a **team plugin for Claude Code** that brings the same standard to whoever is coding — skills to create, onboard, audit and operate a stack, plus an MCP server that connects Claude to the team's task tracker.

## Highlights

- **Versioned reusable workflows (`@v1`)**: each app has three short files that call the central ones. CI runs tests against real Postgres and Redis, `ruff`, pending-migration checks, `check --deploy` (pointed at the production settings module when the app splits dev and prod), a smoke test proving the ASGI server (uvicorn) boots and answers the health endpoint, frontend lint and typecheck, image builds, `docker compose config`, `nginx -t` and `gitleaks` over the full history.
- **Destructive-migration gate**: a detector parses the migrations a PR adds or changes (AST analysis) and blocks `RemoveField`, `DeleteModel`, renames, `RunSQL` and edits to old migrations until the PR gets an approval label.
- **Images on GHCR tagged with the commit SHA**, published in parallel with the tests. The deploy waits for both and only proceeds if the commit **is still the latest on `main`** — a late deploy can never put an old version on top of a newer one.
- **Deploys through a self-hosted runner** that calls a single, narrowly privileged script: it verifies the commit is on `main` via the GitHub API, pulls the images, backs up the database, brings the stack up, migrates, runs `collectstatic`, health-checks and **rolls back to the previous version on its own if anything fails**.
- **Manual rollback by SHA**: one button in Actions; if the image already exists in the registry, nothing is rebuilt.
- **Automated house-standard check**: compose, Dockerfiles, nginx, an `.env.example` with no values, `DEBUG` read from the environment — errors block the PR; anything that slows builds or deploys becomes a warning linking to the fix. Newly onboarded stacks run in "adoption mode" (warnings only).
- **Measured optimization guide**: on the pilot app, merge-to-live time went from about 20 min to 1 min 43 s; the frontend image from 1.39 GB to 357 MB; the test suite from 172 s to 13 s (figures recorded in the project's documentation).
- **Claude Code plugin** with 6 skills (app standard, CI/CD onboarding, deploy, stack audit, incident runbooks, central sign-in integration) and an **MCP server with 45 tools** on top of the task tracker's API — tasks, checklists, comments, time tracking, personal board, delegation and attachments — using each person's own token.

## Screens

This is an infrastructure project, so there are no screens. The showcase relies on diagrams and an illustrative configuration snippet.

## Architecture — from PR to production

```mermaid
flowchart TD
    pr["Developer opens a PR"] --> ci["Reusable CI on GitHub-hosted runners<br/>ruff · migrations · tests on Postgres<br/>lint · typecheck · image build<br/>house standard · gitleaks"]
    pr --> mig{"Destructive migration?"}
    mig -- "yes, no label" --> bloq["PR blocked"]
    mig -- "no, or approved" --> merge
    ci --> merge["Merge into main"]
    merge --> ci2["CI on main<br/>(tests, lint, standard)"]
    merge --> pub["Publish to GHCR<br/>build + push, tag = SHA"]
    ci2 --> ult{"Still the latest<br/>commit on main?"}
    pub --> ult
    ult -- "no" --> pula["Skip: the newer commit's<br/>deploy carries this one"]
    ult -- "yes" --> dep["Deploy via self-hosted runner<br/>backup → up → migrate → health"]
    dep --> ok{"Healthy?"}
    ok -- "yes" --> noar["Live<br/>prune old images"]
    ok -- "no" --> volta["Automatic rollback<br/>to the previous version"]
    manual["Manual rollback<br/>(Run workflow with a SHA)"] --> pub2["Publish<br/>(image exists: no rebuild)"]
    pub2 --> dep
```

Unreviewed PR code only ever runs on GitHub-hosted runners. The self-hosted runner only receives deploy jobs from `main` and, even then, can run exactly one command: "put this SHA of this stack live".

## Deploy on the runner, step by step

```mermaid
sequenceDiagram
    autonumber
    participant GH as GitHub Actions
    participant R as Self-hosted runner
    participant S as Deploy script (restricted sudo)
    participant API as GitHub API
    participant REG as GHCR
    participant DB as Postgres
    participant DC as Docker Compose

    GH->>R: deploy job (stack, SHA), one per stack at a time
    R->>R: server compose and nginx match the commit?
    Note over R: mismatch: stop, show the diff, ask for human review
    R->>S: stack + SHA, job token over stdin
    S->>API: compare main with the SHA
    API-->>S: identical or behind (it is on main)
    S->>REG: log in with a throwaway config and pull the SHA images
    alt critical stack (manual approval)
        S-->>R: left pending, the admin approves on the server
    else automatic deploy
        S->>DB: pre-deploy pg_dump (checks the dump header)
        S->>DC: up --wait with the new tag
        S->>DC: migrate and collectstatic
        S->>DC: health check with retries
        alt any step failed
            S->>DC: up with the previous tag
            S-->>R: FAILED, rolled back to the previous version
        else all good
            S->>S: record the version and prune old images
            S-->>R: ok, SHA is live
        end
    end
    R->>GH: check the public URL and write the job summary
```

## How each app consumes the platform

```mermaid
flowchart LR
    subgraph central["Central pipeline repository (tag v1)"]
        wci["ci.yml"]
        wpub["publicar.yml"]
        wdep["deploy.yml"]
        wcd["cd.yml"]
        wmig["migracoes.yml"]
        apad["action: house standard"]
        amig["action: migration detector"]
        wci --> apad
        wmig --> amig
    end

    subgraph app["Each app repository"]
        aci["short ci.yml<br/>(PR and push to main)"]
        acd["cd.yml<br/>(manual rollback)"]
        amg["migracoes.yml<br/>(reacts to the label)"]
    end

    subgraph plugin["Team Claude Code plugin"]
        sk1["skill: app standard"]
        sk2["skill: CI/CD onboarding"]
        sk3["skill: deploy"]
        sk4["skill: stack audit"]
        sk5["skill: runbooks"]
        sk6["skill: central sign-in"]
        mcp["Task tracker MCP<br/>(45 tools)"]
    end

    aci --> wci
    aci --> wpub
    aci --> wdep
    acd --> wcd
    amg --> wmig
    wdep --> runner["Self-hosted runner<br/>and deploy script"]
    dev["Developer in Claude Code"] --> plugin
    sk2 -- "copies the templates and fills in with:" --> app
    sk4 -.->|same rules, manual version| apad
    mcp -- "public API + personal token" --> tarefas["Task tracker"]
```

An app adopts the standard by copying three template workflows and filling in `with:`. An illustrative snippet (names replaced with placeholders; input names are in Portuguese, as in the original):

```yaml
jobs:
  ci:
    uses: <org>/<pipelines-repo>/.github/workflows/ci.yml@v1
    with:
      app: my-app
      dominio: my-app.example.com
      construir-imagens: ${{ github.event_name != 'push' }}  # on main, `publicar` builds
      imagens: >-
        [{"servico": "backend",  "contexto": "backend"},
         {"servico": "frontend", "contexto": "frontend"}]

  publicar:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: <org>/<pipelines-repo>/.github/workflows/publicar.yml@v1
    permissions: { contents: read, packages: write }
    with:
      app: my-app
      imagens: >-
        [{"servico": "backend",  "contexto": "backend"},
         {"servico": "frontend", "contexto": "frontend"}]

  deploy:
    needs: [ci, publicar]
    uses: <org>/<pipelines-repo>/.github/workflows/deploy.yml@v1
    permissions: { contents: read, packages: read }
    with:
      app: my-app
      sha: ${{ needs.publicar.outputs.sha }}
```

## Stack

| Layer | Technologies |
|---|---|
| Pipelines | GitHub Actions (reusable workflows and composite actions), actions pinned by SHA |
| CI | Postgres and Redis as services, ruff, uvicorn (ASGI boot check), ESLint/TypeScript, Docker Buildx with `gha` cache, gitleaks, actionlint and shellcheck (self-test) |
| Registry | GitHub Container Registry, tag = full commit SHA |
| Deploy | Self-hosted runner, Bash script with `flock`, Docker Compose (`up --wait`), `pg_dump` |
| Checks | Python (PyYAML, `ast`) for the house standard and the migration detector |
| Maintenance | Dependabot for base-image digests only |
| Developer Experience | Claude Code plugin (private marketplace), Markdown skills, Python MCP server (MCP SDK + httpx), `SessionStart` hook |

## Technical decisions and challenges

- **The production runner trusts no one, not even its own workflow.** It runs as a user with no Docker access, and sudo allows a single command: `deploy <stack> <sha>`. The script checks via the GitHub API that the SHA is on `main`, receives the job token over stdin (never as an argument, never on disk) and logs in to the registry using a throwaway config. Rolling back, choosing an image and approving pending deploys are admin-only actions.
- **Compose and nginx never change through CD.** The deploy only swaps image tags; if the server's infrastructure files differ from the commit's, the job stops and shows the diff. A PR therefore cannot push `privileged`, a new volume or a new port to production without human review.
- **Publish in parallel with the tests, but ship only the latest commit.** Image builds no longer wait for CI; the deploy waits for both. A separate job checks that the commit is still the latest on `main` before joining the queue — GitHub's queue keeps only the most recent pending job, so a late deploy would otherwise take the newer one's place.
- **Automatic rollback, with its limit stated plainly.** Rolling back swaps the images but does not undo migrations. That is why there is a pre-deploy backup, validated by the dump header (size is useless: on the first deploy the database really is empty), and why destructive migrations are gated at PR time. The rollback also restarts nginx even when `up` fails halfway (nginx only resolves upstreams at startup and would return 502 for recreated containers), and it tells "site down" apart from "site up with an unhealthy auxiliary service".
- **Measure before optimizing.** The optimization guide shows how to time every job and every deploy step, and maps each symptom to a cause: base images without a digest (the server re-downloaded everything), Next.js without `standalone`, production password hashing in tests, healthchecks without `start_interval`. CI now warns about each one.
- **An MCP with no shortcuts.** The MCP server talks only to the task tracker's public API, with the user's personal token, so the same permissions, validations and history as the UI apply. Because Claude Code gives up connecting to an MCP server within seconds, the Python environment is prepared by a hook when the session opens, and the plugin's CI boots the MCP on Python 3.11 and 3.13 and checks the tool list.

## My role

I created both pieces and am the largest contributor to each. In the pipeline repository I wrote 11 of the 23 commits on `main` (8 of 14 excluding merges): the whole v1 (reusable workflows, migration detector, standard check, deploy script, server installer, app templates and docs), the fixes that came out of the pilot, the parallel pipeline (publish alongside tests, deploy only the latest commit), the optimization guide with Dependabot, the ASGI boot check in CI and the nginx-restart fix in the rollback. The remaining commits came from other team members: stack configuration, `pyproject.toml` support, a settings-module option for `check --deploy`, and installer checks. In the Claude Code plugin I wrote 18 of the 22 commits (11 of 13 excluding merges): the marketplace, five of the six skills, the MCP plugin (grown from 34 to 45 tools, adding the personal board, delegation and attachments), the launcher and the CI with a smoke test; the central sign-in integration skill came from another team member.

---

The source code of this project is private; this repository is a showcase only. Contact: [github.com/roger-mont](https://github.com/roger-mont).
