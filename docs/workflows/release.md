# Release workflow (usage)

`release-reusable.yml` is a reusable, callable workflow that automates publishing artifacts and creating a GitHub Release whenever a caller repository pushes a version tag (`v*`).

It automatically detects which artifact types exist in the calling repository and **only runs the relevant publish jobs**:

| Artifact | Trigger condition | Publish target |
|---|---|---|
| npm package | `package.json` present | npmjs.com registry |
| Python package | `pyproject.toml` or `setup.py` present | PyPI via Twine |
| Docker image | `Dockerfile` present | Docker registry (default: GHCR) |
| GitHub Release | Always (after other jobs) | GitHub Releases page |

---

## Required secrets

Add these in **Settings → Secrets and variables → Actions** of the calling repository. Only add the ones relevant to your artifact type.

| Secret name | When needed | Description |
|---|---|---|
| `NPM_TOKEN` | npm projects | Automation token from npmjs.com |
| `PYPI_TOKEN` | Python projects | API token from pypi.org |
| `DOCKER_USERNAME` | Docker projects | Registry username (e.g. Docker Hub or GHCR user) |
| `DOCKER_PASSWORD` | Docker projects | Registry password or personal access token |

> **Note:** No extra secrets are needed for the GitHub Release step — it uses the built-in `GITHUB_TOKEN`.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `docker_image` | No | `ghcr.io/<calling-repo>` | Full Docker image name, e.g. `ghcr.io/my-org/my-app` |
| `node_version` | No | `20` | Node.js version used for npm publish |
| `python_version` | No | `3.12` | Python version used for PyPI publish |

---

## Repository visibility requirement

GitHub only allows cross-repository `workflow_call` access to reusable workflows in **public** repositories (or within the same organisation on a paid plan). If `github-reusable` is private, callers in other repositories will receive:

```
workflow was not found
```

To resolve this, make `github-reusable` public: **Settings → General → Danger Zone → Change repository visibility → Make public**. The workflows contain no secrets or sensitive logic — they are safe to be public.

## How to call from another repository

Create `.github/workflows/release.yml` in your repository:

```yaml
name: Release

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: write
  packages: write

jobs:
  release:
    uses: ngroegli/github-reusable/.github/workflows/release-reusable.yml@main
    with:
      docker_image: ghcr.io/ngroegli/my-app   # optional override
    secrets:
      npm_token: ${{ secrets.NPM_TOKEN }}
      pypi_token: ${{ secrets.PYPI_TOKEN }}
      docker_username: ${{ secrets.DOCKER_USERNAME }}
      docker_password: ${{ secrets.DOCKER_PASSWORD }}
```

> The `push: tags: v*` trigger lives in **your** repository's caller workflow, not in the reusable workflow. This keeps the reusable workflow generic and lets each project choose its own tag pattern.

---

## Job execution order

```
detect
  ├── publish-npm     (skipped if no package.json)
  ├── publish-pypi    (skipped if no pyproject.toml / setup.py)
  └── publish-docker  (skipped if no Dockerfile)
        └── github-release  (always runs if detect succeeded)
```

All three publish jobs run **in parallel** after `detect` finishes. The `github-release` job waits for all of them (it still runs even if some were skipped).

---

## Version badge

After each release, the workflow automatically updates a version badge in the calling repo's `README.md` — provided a placeholder comment exists.

**Add this to your `README.md` once** (the workflow keeps the badge line up to date automatically):

```markdown
<!-- release-badge -->
[![Latest release](https://img.shields.io/github/v/release/ngroegli/my-repo?label=release)](https://github.com/ngroegli/my-repo/releases/latest)
```

Replace `ngroegli/my-repo` with your own `owner/repo`.

If the placeholder is absent, the badge step is a no-op — nothing is committed and no error is raised. The step summary always shows the correct badge snippet regardless.

> The badge is powered by [shields.io](https://shields.io) and reflects the latest GitHub Release. No API key or additional setup is needed.

---

## Notes

- The Docker image is tagged with both the version tag (e.g. `v1.2.3`) and `latest`.
- The GitHub Release is auto-generated from commit messages using `generate_release_notes: true`.
- `requirements.txt` alone does **not** trigger a PyPI publish — it is treated as a dev-dependencies file, not a publishable package. Add a `pyproject.toml` or `setup.py` to enable Python publishing.
- The `softprops/action-gh-release` action is pinned to a specific commit SHA to prevent supply-chain drift.
