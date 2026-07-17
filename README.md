# NextStageSY — Shared CI/CD

Organization-level repository holding **reusable GitHub Actions workflows** for
NextStageSY projects. The goal is a single source of truth: update a workflow
here and every consuming repo inherits the change on its next run.

## Available workflows

| Workflow | Path | Purpose |
| --- | --- | --- |
| **Web CI** | `.github/workflows/web-ci.yml` | Build + test the ABP layered web apps and build their Docker images via `docker compose`. Build only (no image push / deploy yet). |

## Using `web-ci.yml`

Each web app (`<Project>-Web`) has a tiny caller at
`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

# Cancel superseded runs on the same ref.
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  ci:
    uses: NextStageSY/.github/.github/workflows/web-ci.yml@main
```

> The double `.github/.github/` is correct: the org repo is named `.github`,
> and reusable workflows must live in its `.github/workflows/` folder.

### What it does

1. Resolves the project name from the `*.slnx` file (so the caller is identical
   across every app).
2. `dotnet restore` / `build` / `test` the solution (unit tests use an
   in-memory SQLite database — no services required).
3. `dotnet publish` the `HttpApi.Host` and `DbMigrator` projects (ABP's
   publish-then-package convention).
4. `docker compose build` the three images (`api`, `db-migrator`, `angular`)
   defined in the app's `etc/docker-compose/docker-compose.ci.yml`.

### Inputs

| Input | Default | Description |
| --- | --- | --- |
| `dotnet-version` | `10.0.x` | .NET SDK version to install. |
| `compose-file` | `etc/docker-compose/docker-compose.ci.yml` | CI compose file, relative to the repo root. |

## Requirements in a consuming repo

* A `*.slnx` at the repo root whose basename is the project prefix.
* `src/<Project>.HttpApi.Host` and `src/<Project>.DbMigrator` projects with
  their ABP-generated `Dockerfile`s.
* An `angular/` folder with a self-contained `Dockerfile`.
* `etc/docker-compose/docker-compose.ci.yml` wiring those three build contexts.

## Access

This repo is **private**. For its reusable workflows to be callable from the
other (private) org repos, GitHub requires:

**Settings → Actions → General → Access → "Accessible from repositories owned
by the NextStageSY organization".**

## Roadmap

* CD: push images to a registry (e.g. GHCR) and deploy.
* A full `docker-compose.yml` (Postgres + services) for local run / integration
  smoke tests.
* Starter workflow template so new repos scaffold the caller from the GitHub UI.
