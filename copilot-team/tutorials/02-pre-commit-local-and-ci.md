# Tutorial 2: same lint checks locally (pre-commit) and in CI (Azure Pipelines)

**Goal:** if ruff passes on your machine, it passes in CI, and the other way around.
**Time:** 15 minutes per repo, plus 1 minute per developer.

## The problem we're solving

If CI installs one ruff version and pre-commit installs another, the same code can pass locally and fail in CI. You then can't reproduce the failure. This also affects the agent: it fixes code until pre-commit passes, then CI rejects it.

## The rule: one version, one list of checks

| What | Defined once, in | Used by |
|---|---|---|
| Ruff **version** | `pyproject.toml` (dev dependencies, exact pin) | your venv, CI |
| Ruff **rules** | `pyproject.toml` (`[tool.ruff]`) | ruff, wherever it runs |
| **Which checks run** | `.pre-commit-config.yaml` | `git commit` locally, the pipeline step in CI |

Pre-commit uses **local hooks** (`repo: local`, `language: system`). It doesn't install its own ruff. It runs the ruff that's already in your venv, so there is no second version to keep in sync.

## Steps for the repo (once)

### 1. Pin the tools as dev dependencies

`pyproject.toml`:

```toml
[project.optional-dependencies]
dev = [
    "ruff==X.Y.Z",       # exact pin. Bumping it here is the ONLY place to bump it.
    "pytest==X.Y.Z",
    "pre-commit==X.Y.Z",
]

[tool.ruff]
line-length = 100        # team values come from tools/, this is only an example
target-version = "py312"

[tool.ruff.lint]
select = ["E", "F", "I", "B"]
```

> If the repo uses **uv**, `uv add --dev ruff==X.Y.Z` writes the pin and `uv.lock` records it. Same principle.

### 2. Declare the checks

`.pre-commit-config.yaml`:

```yaml
repos:
  - repo: local                      # use tools from our venv, not pre-commit's own copies
    hooks:
      - id: ruff-check
        name: ruff check
        entry: ruff check --fix
        language: system             # = "run the command found on PATH"
        types_or: [python, pyi]
        require_serial: true
      - id: ruff-format
        name: ruff format
        entry: ruff format
        language: system
        types_or: [python, pyi]
```

pytest is **not** in pre-commit: it's too slow to run on every commit. It runs in CI.

### 3. Run the same checks in the pipeline

`azure-pipelines.yml` (only the relevant steps):

```yaml
steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: '3.12'

  - script: pip install -e ".[dev]"
    displayName: Install dev dependencies (ruff version comes from pyproject.toml)

  - script: pre-commit run --all-files --show-diff-on-failure
    displayName: Lint (same hooks as on your machine)

  - script: pytest
    displayName: Unit tests
```

Why CI runs **pre-commit** instead of calling `ruff` directly: the list of checks is then also defined once. Adding a hook to `.pre-commit-config.yaml` adds it to CI automatically. With `language: system`, pre-commit doesn't build any environment, so this step costs only a few seconds.

In CI, `ruff check --fix` and `ruff format` change files on the build agent and the step fails. That's intended: it means someone committed without pre-commit. `--show-diff-on-failure` prints what would have changed.

### 4. Make it mandatory

Azure DevOps → Repos → Branches → `master` → **Branch policies** → **Build validation** → add this pipeline, set **Required**. This is the only check nobody can skip.

## Steps for each developer (once per repo)

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate    Linux/macOS: source .venv/bin/activate
pip install -e ".[dev]"
pre-commit install          # registers the git hook in .git/hooks/pre-commit
pre-commit run --all-files  # optional first run on the whole repo
```

## Known problems

| Symptom | Cause | Fix |
|---|---|---|
| `ruff: command not found` when committing | `language: system` uses PATH, and the venv isn't active (common with the VS Code **Source Control** commit button) | Commit from a terminal where the venv is active, or select the venv as the VS Code Python interpreter. With uv: `entry: uv run ruff check --fix`. |
| Passes locally, fails in CI | Your venv has an old ruff | `pip install -e ".[dev]"` again after a version bump |
| "I committed with `--no-verify`" | Hooks are skippable on purpose | CI catches it. That's why CI is the mandatory gate. |

## Sources
- pre-commit, repository-local hooks: https://pre-commit.com/#repository-local-hooks
- pre-commit, usage in CI: https://pre-commit.com/#usage-in-continuous-integration
- Ruff, pre-commit integration: https://docs.astral.sh/ruff/integrations/#pre-commit
- Azure DevOps, branch policies / build validation: https://learn.microsoft.com/en-us/azure/devops/repos/git/branch-policies
