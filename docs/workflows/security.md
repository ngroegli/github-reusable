# Security workflow (usage)

`security-reusable.yml` is the canonical, reusable security scanning workflow. It is called by other repositories via `workflow_call` and runs five parallel scanner jobs, then creates one GitHub Issue per finding and fails the workflow if any findings are present.

## What it scans

| Scanner | Tool | Trigger condition |
|---|---|---|
| JS/TS security pattern scan | `eslint-plugin-security` | `package.json` present |
| Python security lint | Bandit | `pyproject.toml`, `setup.py`, or `requirements.txt` present |
| Secret detection | gitleaks CLI | Always (full git history) |
| npm dependency audit | `npm audit` | `package.json` present |
| Python dependency audit | `pip-audit` | `pyproject.toml`, `setup.py`, or `requirements.txt` present |

> **Why eslint-plugin-security for JS?** It is lightweight (no build step, no account, MIT licence), runs on the existing Node install, and covers the most common OWASP Top 10 patterns in JS/TS: object injection, non-literal `require`/`fs` calls, unsafe regex, `eval`, missing CSRF protection, and timing attacks. It does not attempt deep semantic analysis — that trade-off is intentional.

> **Why Bandit for Python?** Bandit covers the same class of OWASP patterns for Python: hardcoded credentials, SQL injection, `eval`, insecure crypto, unsafe `subprocess` calls, and more. It is focused, fast, and requires no external service.

> **Why pip-audit and not safety?** `pip-audit` is fully free and open source (PyPA / Google, Apache 2.0) and queries the OSV + PyPI Advisory databases with no API key. `safety` v3+ requires a paid key for complete results.

## Findings and issues

- One GitHub Issue is created per finding, labelled by source (`gitleaks`, `npm-audit`, `pip-audit`, `bandit`, `eslint-security`).
- Duplicate issues (same title, already open) are skipped.
- After all findings are processed, a summary issue (`Security scan summary: N new findings`) is created.
- The workflow exits non-zero (fails) if any new issues were created.

## Tokens and secrets

No additional secrets are required. The workflow uses only the built-in `GITHUB_TOKEN` to create issues via the REST API. `GITHUB_TOKEN` is provided automatically by GitHub Actions.

## Permissions

The top-level and job-level permissions are minimal. No `security-events` permission is required because Semgrep writes its report to a local JSON file rather than uploading to the GitHub Code Scanning API.

```yaml
permissions:
  contents: write  # checkout, read files, and commit badge update
  issues: write    # create issues (reporting job only)
```

The **caller workflow** must also declare `contents: write`. GitHub enforces that a reusable workflow cannot receive more permissions than the caller grants — if the caller only grants `contents: read`, the reusable workflow's `contents: write` is rejected at parse time.

## Security badge

After each scan run, the workflow automatically updates a security badge in the calling repo's `README.md` — provided a placeholder comment exists. The badge shows `passing` (green) when no new findings are created, and `failing` (red) when findings exist.

**Add this to your `README.md` once** (the workflow keeps the badge line up to date automatically):

```markdown
<!-- security-badge -->
[![Security scan](https://img.shields.io/badge/security-passing-brightgreen)](https://github.com/ngroegli/my-repo/actions/workflows/security.yml)
```

Replace `ngroegli/my-repo` with your own `owner/repo`.

If the placeholder is absent, the badge step is a no-op — nothing is committed and no error is raised. The step summary always shows the correct badge snippet regardless.

> The badge uses [shields.io](https://shields.io) static badges. No API key or additional setup is needed.

---

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
  contents: write
  issues: write

jobs:
  security:
    uses: ngroegli/github-reusable/.github/workflows/security-reusable.yml@main
```

## Job execution order

```
eslint   ──┐
            ├── create-issues-and-fail
secret   ──┤
npm      ──┤
pip      ──┤
bandit   ──┘
```

All five scan jobs run in **parallel**. The `create-issues-and-fail` job waits for all of them (`if: always()`) before processing results.

## Notes

- Reports from each scan job are uploaded as workflow artifacts (`eslint-security-report`, `gitleaks-report`, `npm-audit-report`, `pip-audit-report`, `bandit-report`) and downloaded by the final reporting job. All artifacts are retained for 30 days.
- A Markdown summary table is written to the Actions run UI (`$GITHUB_STEP_SUMMARY`) showing pass/fail counts per scanner.
- `eslint-plugin-security` is installed locally into `node_modules` during the scan job and does not conflict with the project's own ESLint version. All 12 security rules are enabled at `warn` level so the scan never blocks the runner but all findings are captured.
- `requirements.txt`, `pyproject.toml`, or `setup.py` all trigger both `pip-audit` and Bandit. The project's dependencies are installed before pip-audit runs so the full transitive dependency tree is audited.
- Bandit skips `B110` (try/except/pass) and `B112` (try/except/continue) which are non-security patterns. It scans at medium severity and above. `node_modules`, `venv`, and `.venv` directories are excluded.
- The gitleaks scan runs with `fetch-depth: 0` to scan the full commit history, not just the latest commit.
