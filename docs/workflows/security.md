# Security workflow (usage)

`security-reusable.yml` is the canonical, reusable security scanning workflow. It is called by other repositories via `workflow_call` and runs four parallel scanner jobs, then creates one GitHub Issue per finding and fails the workflow if any findings are present.

## What it scans

| Scanner | Tool | Trigger condition |
|---|---|---|
| Static code analysis | CodeQL | Always (JavaScript + Python) |
| Secret detection | gitleaks CLI | Always (full git history) |
| npm dependency audit | `npm audit` | `package.json` present |
| Python dependency audit | `pip-audit` | `pyproject.toml` or `setup.py` present |

> **Why pip-audit and not safety?** `pip-audit` is fully free and open source (PyPA / Google, Apache 2.0) and queries the OSV + PyPI Advisory databases with no API key. `safety` v3+ requires a paid key for complete results.

## Findings and issues

- One GitHub Issue is created per finding, labelled by source (`gitleaks`, `npm-audit`, `pip-audit`, `codeql`).
- Duplicate issues (same title, already open) are skipped.
- After all findings are processed, a summary issue (`Security scan summary: N new findings`) is created.
- The workflow exits non-zero (fails) if any new issues were created.

## Tokens and secrets

No additional secrets are required. The workflow uses only the built-in `GITHUB_TOKEN` to:
- Create issues via the REST API.
- Read Code Scanning (CodeQL) alerts via the `code-scanning` API.

`GITHUB_TOKEN` is provided automatically by GitHub Actions.

## Permissions

The reusable workflow declares these permissions:

```yaml
permissions:
  contents: read      # checkout and read files
  issues: write       # create issues
  security-events: read  # read CodeQL code-scanning alerts
```

The **caller workflow** must grant the same permissions to its `GITHUB_TOKEN`.

## Repository visibility requirement

GitHub only allows cross-repository `workflow_call` access to reusable workflows in **public** repositories (or within the same organisation on a paid plan). If `github-reusable` is private, callers in other repositories will receive:

```
workflow was not found
```

To resolve this, make `github-reusable` public: **Settings → General → Danger Zone → Change repository visibility → Make public**. The workflows contain no secrets or sensitive logic — they are safe to be public.

## How to call from another repository

Create `.github/workflows/security.yml` in your repository:

```yaml
name: Security scan

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

## Job execution order

```
codeql ──┐
         ├── create-issues-and-fail
secret ──┤
npm    ──┤
pip    ──┘
```

All four scan jobs run in **parallel**. The `create-issues-and-fail` job waits for all of them (`if: always()`) before processing results.

## Notes

- Reports from each scan job are uploaded as workflow artifacts (`gitleaks-report`, `npm-audit-report`, `pip-audit-report`) and downloaded by the final reporting job.
- `requirements.txt` alone does **not** trigger `pip-audit` — it is treated as a runtime/dev-deps file, not a package descriptor. Add `pyproject.toml` or `setup.py` to enable Python scanning.
- The gitleaks scan runs with `fetch-depth: 0` to scan the full commit history, not just the latest commit.
