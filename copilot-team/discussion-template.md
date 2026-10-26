# Copilot team practices: draft v0 for discussion

> This is a **draft, expected to be wrong**. Its job is to give us something concrete to react to.
> Goal: share what works without changing how each person likes to work.

## Before the meeting (10 min, async)

Drop your current setup in `tools/inbox/<your-name>/`, as it is, without cleaning it up:
- `copilot-instructions.md`, any `*.instructions.md`, `*.prompt.md`, `*.agent.md`, `SKILL.md`, hook files
- your VS Code `settings.json` lines starting with `chat.` or `github.copilot.`
- nothing at all is also a valid answer: write "I use the defaults"

<!-- Why: we collect real configs before the meeting, so the discussion starts from what people
     actually use, not only from the draft (reduces anchoring on the draft). -->

## The draft, one point at a time

For each point, answer two questions:
- **A. Does this match what you do today?** (yes / partly / no + one line)
- **B. Where would it slow you down?** (a concrete situation, not "it depends")

| # | Proposal | Why | A | B |
|---|---|---|---|---|
| 1 | One shared foundation for all repos (coding style, naming). Each project adds its own instructions: the project's purpose, tech stack, and expected outcome. | Code generated in different repos, by different people, should look like it came from the same team. The project instructions hold what the foundation can't know. | | |
| 2 | A `tools` repo holds skills, instructions and hooks that are not project-specific. Anyone can contribute through a PR. | One place to find and improve shared material, instead of everyone rebuilding the same thing. | | |
| 3 | `tools` also holds the "start a new project" instructions and the project documentation structure. | New projects start from the same base, so they stay consistent over time. | | |
| 4 | How-tos for configuring VS Code (pointing to `tools`, pre-commit, hooks, custom agents). | _to fill_ | | |
| 5 | A written plan → execute → review workflow for features/PBIs, made stronger without replacing how people work now. | _to fill_ | | |

<!-- Why separate A and B: A gives the gap between the draft and reality; B gives the cost of closing it.
     A point that gets "no" on A but nothing on B is cheap to adopt. "Yes" on A with a strong B means
     the point is already done informally and would only add friction. -->

## One recent example (5 min)

Think of **the last time** Copilot gave you code you had to rewrite or throw away.
- What did you ask for?
- What did you get?
- What did you change, and why?

<!-- Why "the last time": people describe a specific recent event more accurately than their general
     habits (Flanagan, 1954, critical incident technique). These stories show tricks and problems
     that the draft didn't anticipate. -->

## Pre-mortem (10 min, together)

> It's six months from now. We adopted this setup and it failed: people stopped using it, or the generated code is still inconsistent.
> **Write down, alone, 2 reasons why it failed.** Then we read them out loud, one per person, until none are left.

<!-- Why: imagining that the plan already failed makes it easier to object, even to the person who
     wrote the draft (Klein, HBR, 2007). Writing alone first stops the first speaker from setting
     the direction. -->

## Output of the meeting

- For each point: keep / change / drop, plus one owner.
- Open questions → issues in the `tools` repo.
