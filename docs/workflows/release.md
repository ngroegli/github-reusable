# Release workflow (usage)

`release-reusable.yml` is a reusable, callable workflow that builds downloadable artifacts and creates a GitHub Release whenever a caller repository pushes a version tag (`v*`). Built artifacts are attached directly to the release page so they can be downloaded from the Releases tab.

It automatically detects which project types exist in the calling repository and builds only what is relevant. Publishing to npm, PyPI, or Docker registries is **not** performed — that is the responsibility of each repository's own deploy workflow.

---

## What gets built and attached

| Project type | Trigger condition | Artifact attached to Release |
|---|---|---|
| npm | `package.json` present | `<name>-<version>.tgz` (npm pack tarball) |
| Python | `pyproject.toml` or `setup.py` present | `<name>-<version>-*.whl` + `<name>-<version>.tar.gz` |
| Docker | `Dockerfile` present | `<repo>-<version>.tar.gz` (docker save image archive) |

---

## Secrets

No secrets are required. The workflow uses only the built-in `GITHUB_TOKEN`, which GitHub injects automatically into every workflow run.

---

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `node_version` | No | `20` | Node.js version used to build npm packages |
| `python_version` | No | `3.12` | Python version used to build Python packages |

---

## Repository visibility requirement

GitHub only allows cross-repository `workflow_call` access to reusable workflows in **public** repositories (or within the same organisation on a paid plan). If `github-reusable` is private, callers in other repositories will receive:

```
workflow was not found
```

To resolve this, make `github-reusable` public: **Settings → General → Danger Zone → Change repository visibility → Make public**. The workflows contain no secrets or sensitive logic — they are safe to be public.

---

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

jobs:
  release:
    uses: ngroegli/github-reusable/.github/workflows/release-reusable.yml@main
    with:
      node_version: '22'      # optional
      python_version: '3.13'  # optional
```

> The `push: tags: v*` trigger lives in **your** repository's caller workflow, not in the reusable workflow. This keeps the reusable workflow generic and lets each project choose its own tag pattern.

---

## Job execution order

```
detect
  ├── build-npm     (skipped if no package.json)
  ├── build-python  (skipped if no pyproject.toml / setup.py)
  └── build-docker  (skipped if no Dockerfile)
        └── github-release  (always runs if detect succeeded)
```

All three build jobs run in parallel after `detect`. The `github-release` job waits for all of them and attaches every artifact they produced.

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

- The GitHub Release is auto-generated from commit messages using `generate_release_notes: true`.
- The Docker artifact is a `docker save` export — a portable image archive, not a registry push. It can be loaded on any machine with `docker load -i <file>.tar.gz`.
- Publishing artifacts (npm, PyPI, Docker registries) must be handled in your own repository's deploy workflow, separate from this release workflow.
- The `softprops/action-gh-release` action is pinned to a specific commit SHA to prevent supply-chain drift.
