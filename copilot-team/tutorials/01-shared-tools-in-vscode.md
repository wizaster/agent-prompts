# Tutorial 1: use the shared `tools` repo (instructions, skills, agents) in VS Code

**Goal:** Copilot in VS Code uses the team's shared instructions, skills and agents in every repo, without copying them into each repo.
**Time:** 10 minutes, once per machine.

## How it works (read this first)

VS Code looks for Copilot customization files in a list of folders. Each list is a setting:

| Setting | What it finds | Defaults include |
|---|---|---|
| `chat.instructionsFilesLocations` | `*.instructions.md` | `.github/instructions`, `~/.copilot/instructions` |
| `chat.agentSkillsLocations` | `<skill-name>/SKILL.md` | `.github/skills`, `~/.copilot/skills` |
| `chat.agentFilesLocations` | `*.agent.md` | `.github/agents`, `~/.copilot/agents` |

We **add our `tools` clone to each list**. Paths starting with `.` are relative to the open repo. Paths starting with `~` are in your home folder. The repo's own files keep working: project rules in the repo + team rules from `tools`.

> VS Code combines all the files it finds. There is no guaranteed order between them, so a repo file and a `tools` file **must not contradict each other**.

## Layout of the `tools` repo

```
tools/
├── instructions/        # team-wide rules, loaded by file type through applyTo
│   ├── python.instructions.md
│   ├── sql.instructions.md
│   └── tests.instructions.md
├── skills/              # one folder per skill, each with a SKILL.md
│   └── <skill-name>/SKILL.md
├── agents/              # *.agent.md (e.g. planner, reviewer)
├── hooks/               # hook files and scripts to copy into repos (see tutorial 3)
└── docs/                # these tutorials
```

## Steps

### 1. Clone `tools` to a fixed place

```bash
git clone <azdo-url>/tools ~/repos/tools
```

Use `~/repos/tools` unless you have a reason not to. The setup check (tutorial 3) looks there by default.

### 2. Point VS Code to it

Open **user** settings as JSON (`Ctrl+Shift+P` → *Preferences: Open User Settings (JSON)*) and add:

```jsonc
{
  // Keep the default folders AND add ours. If you only write ours, you may lose the defaults.
  "chat.instructionsFilesLocations": {
    ".github/instructions": true,
    "~/.copilot/instructions": true,
    "~/repos/tools/instructions": true
  },
  "chat.agentSkillsLocations": {
    ".github/skills": true,
    "~/.copilot/skills": true,
    "~/repos/tools/skills": true
  },
  "chat.agentFilesLocations": {
    ".github/agents": true,
    "~/.copilot/agents": true,
    "~/repos/tools/agents": true
  }
}
```

Use **user** settings, not workspace settings. Workspace settings are committed to a repo, and your clone path isn't the same as your colleague's.

### 3. (If you cloned somewhere else) set `TEAM_TOOLS_DIR`

Use the same path as in step 2. The setup check reads this variable.

```powershell
# Windows (then restart VS Code)
setx TEAM_TOOLS_DIR "D:\repos\tools"
```

```bash
# Linux/macOS: add to ~/.bashrc or ~/.zshrc
export TEAM_TOOLS_DIR="$HOME/code/tools"
```

### 4. Check that it works

1. Open any Python repo and open Copilot Chat in agent mode.
2. Ask something about a `.py` file.
3. Expand **References** under the answer. `python.instructions.md` from `tools` should be listed.
4. Type `/` in the chat. The team skills should appear in the list.

## Keeping it up to date

- `cd ~/repos/tools && git pull`. Nothing else is needed: VS Code reads the files from the folder.
- The session-start check (tutorial 3) warns you when your clone is behind `master`.
- Contributing: create a branch in your clone, then open a PR. While you're on that branch, **you** are testing your change. Everyone else gets it after the merge.

## Known limits

- **Copilot CLI doesn't read VS Code settings.** If you use the CLI, copy or link the folders into `~/.copilot/instructions`, `~/.copilot/skills` and `~/.copilot/agents`.
- Nothing in the repo records which version of `tools` produced the code. Open point: a `Tools-Ref:` commit trailer.
- Skills are loaded when the model judges them relevant (from their `description`), or when you type `/skill-name`. Discovery doesn't guarantee use.

## Sources
- VS Code, custom instructions: https://code.visualstudio.com/docs/copilot/customization/custom-instructions
- VS Code, agent skills: https://code.visualstudio.com/docs/copilot/customization/agent-skills
- VS Code, custom agents: https://code.visualstudio.com/docs/copilot/customization/custom-agents
- VS Code, Copilot settings reference: https://code.visualstudio.com/docs/copilot/reference/copilot-settings
