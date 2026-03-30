# GitHub Actions - reusable

This private repository stores reusable GitHub Actions workflows and composite actions used across projects.

## Available workflows

| Workflow | File | Description | Docs |
|---|---|---|---|
| Security scan | `.github/workflows/security-reusable.yml` | CodeQL, gitleaks, npm audit, pip-audit. Creates one issue per finding and fails the run. | [docs/workflows/security.md](docs/workflows/security.md) |
| Release & publish | `.github/workflows/release-reusable.yml` | Detects npm / Python / Docker projects and publishes on tag push. Creates a GitHub Release. | [docs/workflows/release.md](docs/workflows/release.md) |

## Quick usage

**Security scan** — add to your repo as `.github/workflows/security.yml`:
```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

permissions:
  contents: read
  issues: write
  security-events: read

jobs:
  security:
    uses: ngroegli/github-reusable/.github/workflows/security-reusable.yml@main
```

**Release** — add to your repo as `.github/workflows/release.yml`:
```yaml
on:
  push:
    tags: [ 'v*' ]

permissions:
  contents: write
  packages: write

jobs:
  release:
    uses: ngroegli/github-reusable/.github/workflows/release-reusable.yml@main
    secrets:
      npm_token: ${{ secrets.NPM_TOKEN }}
      pypi_token: ${{ secrets.PYPI_TOKEN }}
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_password: ${{ secrets.DOCKER_PASSWORD }}
```

See the `docs/` folder for full input/secret references and detailed usage notes.
