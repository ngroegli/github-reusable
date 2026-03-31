# Security workflow (usage)

`security-reusable.yml` is the canonical, reusable security scanning workflow. It is called by other repositories via `workflow_call` and runs four parallel scanner jobs, then creates one GitHub Issue per finding and fails the workflow if any findings are present.

## What it scans

| Scanner | Tool | Trigger condition |
|---|---|---|
| Static code analysis | CodeQL | Always (JavaScript + Python) |
| Python code security lint | Bandit | `pyproject.toml`, `setup.py`, or `requirements.txt` present |
| Secret detection | gitleaks CLI | Always (full git history) |
| npm dependency audit | `npm audit` | `package.json` present |
| Python dependency audit | `pip-audit` | `pyproject.toml`, `setup.py`, or `requirements.txt` present |

> **Why pip-audit and not safety?** `pip-audit` is fully free and open source (PyPA / Google, Apache 2.0) and queries the OSV + PyPI Advisory databases with no API key. `safety` v3+ requires a paid key for complete results.

> **Why Bandit alongside CodeQL?** Bandit is a dedicated Python security linter that catches Python-specific anti-patterns (hardcoded credentials, `eval`, insecure `subprocess` calls, SQL injection, etc.) with high precision. CodeQL provides broader cross-language semantic analysis; the two tools are complementary.

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
  contents: read          # checkout and read files
  issues: write           # create issues
  security-events: write  # CodeQL: upload SARIF results to Code Scanning
  actions: read           # CodeQL: read workflow run metadata for SARIF correlation
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
  # Scan on every push and pull request to main.
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
  # Weekly scheduled scan (every Monday at 02:00 UTC) to catch
  # newly published vulnerabilities in unchanged code.
  schedule:
    - cron: "0 2 * * 1"
  # Allow manual triggering from the Actions UI.
  workflow_dispatch:

permissions:
  contents: read
  issues: write
  security-events: write
  actions: read

jobs:
  security:
    uses: ngroegli/github-reusable/.github/workflows/security-reusable.yml@main
```

## Job execution order

```
codeql  ──┐
           ├── create-issues-and-fail
secret  ──┤
npm     ──┤
pip     ──┤
bandit  ──┘
```

All five scan jobs run in **parallel**. The `create-issues-and-fail` job waits for all of them (`if: always()`) before processing results.

## Notes

- Reports from each scan job are uploaded as workflow artifacts (`gitleaks-report`, `npm-audit-report`, `pip-audit-report`, `bandit-report`) and downloaded by the final reporting job. All artifacts are retained for 30 days.
- A Markdown summary table is written to the Actions run UI (`$GITHUB_STEP_SUMMARY`) showing pass/fail counts per scanner.
- `requirements.txt`, `pyproject.toml`, or `setup.py` all trigger both `pip-audit` and Bandit. The project's dependencies are installed before pip-audit runs so the full transitive dependency tree is audited.
- Bandit skips `B110` (try/except/pass) and `B112` (try/except/continue) which are non-security patterns. It scans at medium severity and above. `node_modules`, `venv`, and `.venv` directories are excluded.
- The gitleaks scan runs with `fetch-depth: 0` to scan the full commit history, not just the latest commit.
