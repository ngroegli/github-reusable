# GitHub Actions - reusable

[![CI](https://github.com/ngroegli/github-reusable/actions/workflows/ci.yml/badge.svg)](https://github.com/ngroegli/github-reusable/actions)

This repository stores reusable GitHub Actions workflows used across projects.

> **Repository must be public.** GitHub only allows cross-repository `workflow_call` to reusable workflows in public repositories (or within the same paid organisation). If this repository is private, callers in other repositories will fail with `workflow was not found`.

## Available workflows

| Workflow | File | Description | Docs |
|---|---|---|---|
| Security scan | `security-reusable.yml` | Scans for secrets, dependency vulnerabilities, and insecure code patterns. Creates one GitHub Issue per finding and fails the run. | [docs/workflows/security.md](docs/workflows/security.md) |
| Release | `release-reusable.yml` | Builds artifacts (npm tarball, Python wheel, Docker image archive) and creates a GitHub Release on tag push. | [docs/workflows/release.md](docs/workflows/release.md) |

## Quick start

**Security scan** — create `.github/workflows/security.yml` in your repository:

```yaml
name: Security scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: "0 2 * * 1"
  workflow_dispatch:

permissions:
  contents: write
  issues: write

jobs:
  security:
    uses: ngroegli/github-reusable/.github/workflows/security-reusable.yml@main
```

**Release** — create `.github/workflows/release.yml` in your repository:

```yaml
name: Release

on:
  push:
    tags: ['v*']

permissions:
  contents: write

jobs:
  release:
    uses: ngroegli/github-reusable/.github/workflows/release-reusable.yml@main
```

See [docs/workflows/security.md](docs/workflows/security.md) and [docs/workflows/release.md](docs/workflows/release.md) for full documentation including inputs, badge setup, and job execution details.
