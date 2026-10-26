# Copilot team practices: decisions and open points

Reread this before answering or presenting anything, so no item gets dropped.

## Context
- Repos, CI and docs are on Azure DevOps → no GitHub.com features (org instructions, cloud agent, Copilot code review).
- The team is small inside a large org → no org-level Copilot settings. Everything sits at team level.
- Main client: VS Code. Plan: probably Enterprise (to confirm with the admin).
- Principle: enable people, don't impose. Discussion starts from a draft (v0, expected to be wrong).

## Decided
| # | Decision | Reason |
|---|---|---|
| 1 | Shared foundation in a `tools` repo (skills, instructions, hooks, new-project instructions, doc structure). Everyone can contribute through PRs. | One place, everyone can improve it |
| 2 | The foundation reaches each machine through local VS Code settings pointing to a `tools` clone | One copy, cleaner. Accepted cost: everyone has to do the setup. |
| 3 | A session-start hook checks the local config and runs `git fetch` on `tools`, then **warns** if behind (no auto pull) | _reason to write_ |
| 4 | Same hook checks pre-commit is installed, otherwise points to the doc | Nudges the setup without blocking |
| 5 | Ruff pinned once, as a dev dependency (lock file). Pre-commit uses a `local` hook. CI installs dev deps. (option 2) | Local and CI results match |
| 6 | CI + branch policy = the only mandatory gate. Hooks and pre-commit = earlier, cheaper feedback. | Only CI can't be skipped |
| 7 | Stages: **spike** (not business-mandated, no tests), **PoC** (business-mandated, dev/sandbox, hardcoded config allowed, tests only if asked), **prod** (maintained long term) | How much testing depends on the stage |

## Prod checklist (draft)
- [ ] There is a unit test suite
- [ ] All configuration is env-driven, nothing hardcoded in the code logic
- [ ] Integration deployment and testing are documented

## Open
- [ ] Map each bootstrap-skill input → file + location (repo or `tools`). **Start of next session.**
- [ ] Promotion PoC → prod: who decides, when the checklist is run
- [ ] Optional: a `Tools-Ref:` commit trailer (added by a `prepare-commit-msg` hook)
- [ ] Point 5: plan → implement → review workflow (custom agents with `handoffs`)
- [ ] Fill the "Why" cells for points 4 and 5 in `discussion-template.md`

## Deliverables queue
1. ✅ `discussion-template.md`
2. ⏳ Bootstrap skill (after the mapping)
3. ✅ `tutorials/01-shared-tools-in-vscode.md`
4. ✅ `tutorials/02-pre-commit-local-and-ci.md`
5. ✅ `tutorials/03-session-start-hook.md` (script tested locally)
