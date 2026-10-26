# Tutorial 3: a session-start hook that checks your setup

**Goal:** when you start a Copilot agent session in a repo, a script checks that:
1. the `tools` repo is cloned,
2. your clone isn't behind `master`,
3. pre-commit is installed in this repo.

If something is missing, the agent tells you and points to the right tutorial. It **never blocks** you and **never changes** anything on your machine.

**Time:** 5 minutes per repo (the bootstrap skill will do it for new repos).

## How it works (read this first)

- A **hook** is a command that Copilot runs at a fixed moment of the agent's work. It's a script, so it runs the same way every time, unlike an instruction that the model may or may not follow.
- Copilot reads hook files from `.github/hooks/*.json` in the repo (or `~/.copilot/hooks/` for personal hooks).
- At `sessionStart`, the only thing the script can return is `additionalContext`: text added to the **agent's** context. The hook **can't show a message to you directly**. It tells the agent, and the agent tells you. The check itself is deterministic. How the agent words the message is not.
- Hooks are a **Preview** feature, and the enterprise can restrict them. If hooks are disabled, nothing happens and nothing breaks.

## Steps

### 1. The hook file

`.github/hooks/session-start.json`:

```json
{
  "version": 1,
  "hooks": {
    "sessionStart": [
      {
        "type": "command",
        "command": "python .github/hooks/check_setup.py",
        "timeoutSec": 15
      }
    ]
  }
}
```

`command` works on Windows and Linux. The script is in Python because everyone on the team reads Python, and it avoids keeping a bash and a PowerShell version.

### 2. The script

`.github/hooks/check_setup.py` (the master copy lives in `tools/hooks/`):

```python
"""Session-start check: warns about missing team setup. Never blocks, never modifies anything."""

import json
import os
import subprocess
from pathlib import Path

TOOLS_DIR = Path(os.environ.get("TEAM_TOOLS_DIR", Path.home() / "repos" / "tools"))
MAIN_BRANCH = "master"
DOCS = "tools/docs (or the Azure DevOps wiki page 'Copilot setup')"


def git(*args: str, cwd: Path | None = None) -> subprocess.CompletedProcess:
    return subprocess.run(["git", *args], cwd=cwd, capture_output=True, text=True, timeout=10)


def check_tools_clone() -> list[str]:
    if not (TOOLS_DIR / ".git").exists():
        return [f"The team tools repo was not found at {TOOLS_DIR}. See tutorial 1 in {DOCS}."]
    try:
        git("fetch", "--quiet", "origin", MAIN_BRANCH, cwd=TOOLS_DIR)
    except subprocess.TimeoutExpired:
        return ["Could not reach the tools repo remote to check for updates (timeout)."]
    behind = git("rev-list", "--count", f"HEAD..origin/{MAIN_BRANCH}", cwd=TOOLS_DIR)
    count = behind.stdout.strip()
    if behind.returncode == 0 and count not in ("", "0"):
        return [f"The local tools repo is {count} commit(s) behind {MAIN_BRANCH}. Run `git pull` in {TOOLS_DIR}."]
    return []


def check_pre_commit() -> list[str]:
    hook_path = git("rev-parse", "--git-path", "hooks/pre-commit").stdout.strip()
    if not hook_path or not Path(hook_path).exists():
        return [f"pre-commit is not installed in this repo. Run `pre-commit install` (tutorial 2 in {DOCS})."]
    return []


def main() -> None:
    problems: list[str] = []
    for check in (check_tools_clone, check_pre_commit):
        try:
            problems += check()
        except Exception as exc:  # a broken check must never break the session
            problems.append(f"Setup check '{check.__name__}' failed to run: {exc}")

    if not problems:
        print("{}")
        return
    message = (
        "At the start of your first reply, tell the user about these setup issues, "
        "in one short list, then continue with their request:\n- " + "\n- ".join(problems)
    )
    print(json.dumps({"additionalContext": message}))


if __name__ == "__main__":
    main()
```

### 3. Test it without Copilot

```bash
python .github/hooks/check_setup.py
```

- Everything fine → prints `{}`
- Something missing → prints `{"additionalContext": "..."}` listing the issues

Then start an agent session in VS Code. If there's an issue, the first reply should mention it.

## Design choices (be ready to explain these)

| Choice | Why | Cost |
|---|---|---|
| `git fetch` + warn, no `git pull` | Someone contributing to `tools` on a branch would see their work changed at every session start | People can ignore the warning |
| Always prints valid JSON and exits normally | A broken check must not stop anyone from working | A broken check can go unnoticed (it's reported as "failed to run") |
| `fetch` limited to 10 s | Offline or VPN off → the session starts anyway | No update check when offline |
| Script copied into each repo | The hook file needs a path that exists in every clone | Copies can drift. Update them from `tools/hooks/` (later: done by the bootstrap skill). |

## Known limits

- It can't check your VS Code settings (tutorial 1, step 2). It only checks that the clone exists. The References check in tutorial 1 covers that.
- It only runs for **agent** sessions, not for code you write yourself. pre-commit and CI cover that.

## Sources
- VS Code, agent hooks: https://code.visualstudio.com/docs/copilot/customization/hooks
- GitHub, hooks configuration reference (fields, `sessionStart` output): https://docs.github.com/en/copilot/reference/hooks-configuration
- git, `rev-parse --git-path`: https://git-scm.com/docs/git-rev-parse
