---
applyTo: "**/.github/workflows/*.yml,**/.github/workflows/*.yaml,**/actions/**/*.yml,**/actions/**/*.yaml,**/actions/**/action.yml,**/actions/**/action.yaml,**/*.md,**/*.d2"
---

Repository-level Copilot instructions — GitHub Actions Reusable Workflows and Custom Actions

Short summary
These instructions apply to any repository that contains GitHub Actions workflows, reusable workflows, composite actions, or JavaScript/Docker actions — whether that is a dedicated shared-library repo or a project repo with its own CI. The primary goal is consistency, security, and maintainability: every workflow and action must be self-contained, pinned to exact versions, clearly documented, and safe to run without introducing supply-chain or secret-leakage risks.

## Repository structure

GitHub imposes two hard constraints on where workflows and actions must live:
- Reusable workflows (those with a `workflow_call` trigger) must live under `.github/workflows/` so GitHub can resolve them when a caller references `<org>/<repo>/.github/workflows/<file>.yml@<ref>`.
- Composite and JavaScript/Docker actions must each live in their own directory containing an `action.yml`. The directory path is part of the public reference callers use (e.g. `<org>/<repo>/actions/<name>@<ref>`), so choose directory names carefully and treat them as part of the public API.

Within those constraints, organise the repository as follows:

- `.github/workflows/` — reusable workflows only. No one-off or internal-use-only workflows belong here; they pollute the public surface. Name files to reflect the use case, e.g. `ci-python.yml`, `release-docker.yml`, `lint-pr.yml`.
- `actions/<action-name>/` — one subdirectory per composite or JavaScript/Docker action, each containing an `action.yml` and any supporting code. Keep the directory name short and stable — renaming it is a breaking change for callers.
- `src/` — supporting scripts shared across multiple actions or workflows. Use POSIX sh or Python; avoid other languages unless there is a specific justified reason.
- `docs/` — reference documentation. Every action and every reusable workflow must have a corresponding Markdown page.
- `README.md` at the repo root: purpose, a quick-reference table of every available workflow and action with their required inputs, and links into `docs/`.

```
.github/
  workflows/
    ci-python.yml
    release-docker.yml
actions/
  setup-python-env/
    action.yml
  notify-slack/
    action.yml
src/
  (scripts shared across actions and workflows)
docs/
  workflows/
    ci-python.md
  actions/
    setup-python-env.md
  drawings/
    *.d2
    *.png
README.md
CHANGELOG.md
```

## Reusable workflows

- Declare every reusable workflow with the `workflow_call` trigger. Always expose the caller-facing interface explicitly using `inputs:` and `secrets:` blocks — never rely on inherited environment variables without explicit declaration.
- Provide defaults for every `input` where a sensible default exists. Document each input and secret with a `description` field.
- Mark secrets as `required: true` only when the workflow genuinely cannot run without them. Prefer making secrets optional and guarding the dependent step with an `if:` condition.
- Always pin third-party actions to an exact commit SHA (not a branch or a mutable tag like `v3`). Exception: first-party `actions/*` actions may use a major-version tag (e.g. `actions/checkout@v4`).
- Every job must declare an explicit `runs-on` label. Never use `runs-on: ubuntu-latest` in a reusable workflow — pin to a specific runner label (e.g. `ubuntu-24.04`) so callers get predictable behaviour.
- Set `permissions:` at the workflow level, granting only the minimum permissions required. Default to `read-all` or `contents: read` and elevate only the specific permissions a job needs.
- Always set `timeout-minutes` on every job. Choose a value that reflects the expected duration with a reasonable margin; never omit it.

## Custom actions

- Prefer composite actions over JavaScript or Docker actions when shell commands are sufficient. Use JavaScript actions only when you need cross-platform Node.js logic or fine-grained control over the Actions runtime. Use Docker actions only when a specific tool or binary is unavailable in the runner environment.
- Every `action.yml` must declare `name`, `description`, `author`, `inputs`, `outputs`, and `runs` at the top level. Never omit `description` on any input or output.
- Mark inputs as `required: true` only when there is genuinely no safe default. Always provide a `default` where one makes sense.
- Composite action steps must use `shell: sh` (POSIX) or `shell: python` for Python scripts. Never embed long inline scripts — move them to `src/` and call the script file from the step.
- For JavaScript actions: pin the Node.js version in `runs.using` (e.g. `node20`). Commit the built `dist/` directory to the repository; do not rely on a build step in callers. Use `@actions/core` and `@actions/github` from the official toolkit; do not add unnecessary npm dependencies.
- Always declare every output that a downstream job or caller might consume. Never rely on GITHUB_ENV or GITHUB_OUTPUT side effects that are not declared in `outputs:`.

## Permissions & security

- Never store secrets in workflow files, action files, or any file committed to the repository. All sensitive values must come from `secrets:` inputs or the GitHub secrets store.
- Always set `permissions:` at the workflow level. Start from `contents: read` and add only what is required. Avoid `write-all`.
- When using `GITHUB_TOKEN`, request only the permissions the job actually needs (e.g. `pull-requests: write` for a PR comment step, not `write-all`).
- Pin every third-party action to a full commit SHA. Add a comment on the same line with the tag it corresponds to, e.g.:
  ```yaml
  uses: some-org/some-action@a1b2c3d4e5f6...  # v2.3.1
  ```
- Treat all inputs to reusable workflows and composite actions as untrusted. Never interpolate `${{ inputs.* }}` directly into a `run:` shell command or an `if:` expression that executes code — use an environment variable intermediary to prevent script injection:
  ```yaml
  env:
    INPUT_VALUE: ${{ inputs.some_input }}
  run: echo "$INPUT_VALUE"
  ```
- Do not use `pull_request_target` with `actions/checkout` on the PR head ref unless you fully understand the trust boundary. Prefer `pull_request` for untrusted code.
- Never print secret values with `echo` or `run:` commands. GitHub Actions will attempt to mask them, but relying on masking as the sole protection is not acceptable.

## Error handling & job design

- Every step that produces an artefact or side-effect another step depends on must fail explicitly — do not suppress errors with `|| true` unless the failure is genuinely expected and handled.
- Use `continue-on-error: true` only for non-critical steps where partial failure is acceptable, and always pair it with a subsequent step that reports or surfaces the outcome.
- Group related steps into a single job. Split jobs at natural parallelism boundaries or at trust boundaries (e.g. separate the build job from the deploy job so the deploy job can be gated with an environment protection rule).
- Use `needs:` to express job dependencies explicitly. Never rely on sequential execution by accident.
- Use GitHub Environments (`environment:`) for any job that deploys or has side effects in an external system. This enables protection rules, required reviewers, and audit trails.
- When a job produces outputs consumed by a downstream job, always declare them in `outputs:` on the job and pass them through `needs.<job>.outputs.<key>`.

## Configuration & versioning

- Reusable workflows are versioned by Git tag. Follow semantic versioning (`v1.2.3`). Breaking changes to `inputs` or `secrets` interfaces must increment the major version.
- Maintain a `CHANGELOG.md` at the repo root documenting every released version and its changes.
- When a caller pins to a major-version tag (e.g. `your-org/your-repo/.github/workflows/ci.yml@v1`), that tag must point to the latest stable minor/patch within that major. Create and move the major-version tag on every minor/patch release.
- Never rename or remove an existing `input` or `secret` in a released version without a major version bump. Adding new optional inputs with defaults is a non-breaking change.

## Linting & validation

- All workflow and action YAML files must pass `actionlint` with no errors before being committed. Include an `actionlint` job in this repository's own CI workflow that runs on every push and pull request.
- Use `yamllint` to enforce consistent YAML formatting across all files. Provide a `.yamllint.yml` config at the repo root.
- Validate every action's `action.yml` schema using a JSON Schema validator or the `action-validator` tool in CI.
- Shell scripts in `src/` must pass `shellcheck` with no warnings. Never suppress shellcheck warnings globally — suppress inline only with a documented reason.

## Testing

- Every reusable workflow must have at least one internal caller workflow in this repository (under `.github/workflows/test-*.yml`) that exercises it with representative inputs. These test workflows are for internal CI only — they are not part of the reusable interface and must not have a `workflow_call` trigger.
- Every composite action must have a corresponding test workflow under `.github/workflows/test-<action-name>.yml` that calls the action and asserts its outputs using a subsequent step.
- Use `act` (https://github.com/nektos/act) to run workflow tests locally before pushing. Document the `act` invocation in `docs/testing.md`.
- Test both the happy path and the most critical failure paths (e.g. missing required input, downstream service unavailable).
- Never use a production environment or production secrets in tests. Use dummy secrets and mock service endpoints.

## .gitignore

- The repository must contain a `.gitignore` at the root. At minimum it must exclude:
  - `node_modules/` — npm dependencies for any JavaScript actions.
  - `.env` and any file matching `*.env` — environment variable files.
  - Editor and OS noise: `.DS_Store`, `Thumbs.db`, `.idea/`, `.vscode/` (except shared settings intentionally committed).
- For JavaScript actions: do commit `dist/` (the compiled bundle). Explicitly un-ignore it with `!dist/` if your `.gitignore` would otherwise exclude it.

## When not to follow these instructions

- If a workflow or action is being written for a specific caller repository that has pre-existing conventions that conflict with these instructions, note the deviation in a code comment or ADR rather than silently diverging.
- If GitHub Actions itself changes a platform feature (e.g. deprecates a permission key, runner label, or expression syntax), update these instructions before applying the change to workflows.

## Documentation

- The root of the repository must contain only a `README.md`. Keep it short: repository purpose, a quick-reference table of every available reusable workflow and composite action (name, description, required inputs), and links into `docs/`.
- The `README.md` must include useful badges at the very top (e.g. license, CI status). No other decorative icons or emojis anywhere in any documentation file.
- Never use emojis or decorative icons in any Markdown file — including `README.md`, `docs/`, and inline comments. Plain prose only.
- All detailed documentation lives under `docs/` as Markdown files. Each reusable workflow must have a reference page at `docs/workflows/<name>.md`; each action at `docs/actions/<name>.md`. Each page must document: purpose, all inputs with types and defaults, all outputs, example caller snippet, and any known limitations.
- The repository must contain a `docs/architecture.md` file describing: the overall layout of workflows and actions, how they relate to each other (dependency graph), the versioning strategy, and the trust model (permissions, secret handling). This file must reference the relevant D2 diagrams.
- Architecture, data-flow, and sequence diagrams are written in D2 and stored as `docs/drawings/*.d2`.
- Every `.d2` file must be compiled to a `.png` (same base name, same folder) and committed alongside the source. Compile with `d2 <source>.d2 <source>.png`.
- Reference the compiled `.png` files in Markdown documentation using a standard Markdown image tag with a relative path pointing into `drawings/`.
- Do not commit other binary formats (SVG is acceptable as an alternative export if requested, but PNG is the default).
- When generating or updating a diagram, always update both the `.d2` source and regenerate the `.png`.
