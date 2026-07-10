# steps-re/.github

Shared CI reusable workflows and community-health defaults for every repo
under the `steps-re` GitHub account.

## Why this repo exists

Before this repo, ~30 `steps-re` repos each carried their own copy-pasted
CI workflow. That drifted badly:

- CI hadn't run in months in some repos, so dependency drift (e.g. a NumPy
  2.0 major bump) broke them silently — the failure only surfaced when a PR
  finally touched the repo again.
- A command-injection hole (an untrusted `${{ github.event.pull_request.title }}`
  interpolated straight into a `run:` shell step) existed in one repo's
  workflow, but had already been fixed in a sibling repo. Nobody backported
  the fix because there was nothing to backport it *to*.
- Notification jobs were failing and blocking PRs for unrelated reasons.

This repo is the fix: one reusable workflow that every repo calls, so a fix
made here lands everywhere the next time CI runs. No more hunting down 30
copies of the same YAML.

It also supplies org-wide community-health defaults (PR template) and a
canonical pre-commit config, both meant to be adopted the same way: link to
this repo instead of re-inventing per-repo.

## What's in here

| Path | Purpose |
|---|---|
| `.github/workflows/reusable-python-ci.yml` | The reusable workflow (`workflow_call`). Lint (ruff), test (pytest matrix), secret-scan (gitleaks), and workflow-lint (actionlint + zizmor). |
| `workflow-templates/python-ci.yml` + `.properties.json` | Starter workflow that shows up under a repo's GitHub "Actions → New workflow" picker. It just calls the reusable workflow above. |
| `templates/.pre-commit-config.yaml` | Canonical local pre-commit config (ruff, gitleaks, actionlint, hygiene hooks). |
| `.github/PULL_REQUEST_TEMPLATE.md` | Org-wide default PR template. Applies automatically to any `steps-re` repo that doesn't define its own. |

## Adopting the CI workflow in a repo

Add `.github/workflows/ci.yml` to your repo with:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    # Weekly, Monday 09:00 UTC — catches dependency drift (new major
    # versions of NumPy/etc.) even if nobody touches the repo all week.
    - cron: "0 9 * * 1"
  workflow_dispatch:

permissions:
  contents: read

jobs:
  ci:
    uses: steps-re/.github/.github/workflows/reusable-python-ci.yml@main
    with:
      python-versions: '["3.11","3.12"]'   # optional, this is the default
      run-typecheck: false                  # optional, set true to run mypy if configured
      package-manager: pip                  # optional, pip is currently the only supported value
```

That's it — no `with:` block is required at all if the defaults work for
your repo. Delete lines you don't need to override.

Alternatively, go to **Actions → New workflow** in your repo; the
"Python CI (steps-re shared)" starter template (defined in
`workflow-templates/`) will offer the same file pre-filled.

### What each job does, and why it won't hard-fail on a repo that isn't set up yet

- **lint** — runs `ruff check .` only if it finds a ruff config
  (`[tool.ruff]` in `pyproject.toml`, `ruff.toml`, or `.ruff.toml`).
  Otherwise it logs and passes, so repos without a ruff config aren't
  blocked.
- **test** — installs the package (`pip install -e .[test]`, falling back
  to `pip install -e . && pip install pytest`) and runs pytest across the
  `python-versions` matrix. If there's no `tests/`/`test/` directory or
  `test_*.py` files, it logs and passes instead of failing on "no tests
  collected."
- **secret-scan** — runs `gitleaks/gitleaks-action`. Blocking by design —
  a secret leak should stop the PR.
- **lint-actions** — runs `actionlint` and `zizmor` (which specifically
  catches GitHub Actions workflow-injection patterns, the class of bug that
  motivated this repo) against `.github/workflows/`. Skips gracefully if a
  repo has no `.github/workflows/` at all.

## Adopting the pre-commit config

```bash
curl -o .pre-commit-config.yaml \
  https://raw.githubusercontent.com/steps-re/.github/main/templates/.pre-commit-config.yaml
pip install pre-commit
pre-commit install
pre-commit run --all-files
```

Don't hand-edit a divergent copy per repo — if you need a repo-specific
exception, prefer tool-native ignore config (e.g. `[tool.ruff.lint.per-file-ignores]`
in that repo's own `pyproject.toml`) over forking this file.

## TODOs / known follow-ups

- **SHA-pin third-party actions.** Actions here are pinned to version tags
  (`@v4`, `@v2`, etc.), not commit SHAs. Tags can be moved by the action
  owner; SHA-pinning is the stronger supply-chain guarantee. Nice-to-have,
  not yet done — revisit if we want to harden further (e.g. with
  Dependabot/Renovate configured to bump pinned SHAs).
- **Node/JS CI variant.** This repo currently only covers Python repos.
  If/when a JS or TypeScript repo needs shared CI, add a
  `reusable-node-ci.yml` alongside the Python one using the same pattern.
- **`package-manager` input** currently only supports `pip`. It's exposed
  now so repos can start passing it explicitly; add `uv`/`poetry` branches
  under `test:` in `reusable-python-ci.yml` when a repo needs them, rather
  than each repo forking the workflow.
- Consider making `secret-scan` and `lint-actions` required status checks
  in branch protection once repos have adopted this workflow and a
  baseline of clean scans exists (avoids retroactively blocking merges on
  pre-existing findings).
