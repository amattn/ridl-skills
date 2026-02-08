# ridl-skills

RIDL = Ralph Iteration Definition List

A Claude Code plugin that provides a 3-step pipeline for generating and converting Product Requirements Documents (PRDs) into agent-sized user stories for use in Ralph Loops.


## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **prd** | `/ridl-skills:prd` | Generate a comprehensive PRD with requirements tables, technical architecture, user flows, and release milestones |
| **ridlmd** | `/ridl-skills:ridlmd` | Convert a PRD into `ridl.md` — agent-sized user stories with verifiable acceptance criteria |
| **ridljson** | `/ridl-skills:ridljson` | Convert `ridl.md` into `ridl.json` — the JSON format consumed by a Ralph autonomous loop tool such as chief or ridler|

## Pipeline

The skills form a sequential pipeline. All files live in `projects/[project-name]/`:

```
/ridl-skills:prd      →  prd.md      (comprehensive PRD)
/ridl-skills:ridlmd   →  ridl.md     (agent-sized user stories)
/ridl-skills:ridljson  →  ridl.json   (JSON for autonomous ralph loop execution)
```

## Setup

Add the RIDL marketplace to Claude Code:

```
/plugin marketplace add amattn/ridl-skills
```

Then install the skills:

```
/plugin install ridl-skills@ridl-marketplace
```

Available skills after installation:

- `/ridl-skills:prd` - Generate comprehensive Product Requirements Documents
- `/ridl-skills:ridlmd` - Convert PRDs to agent-sized user stories
- `/ridl-skills:ridljson` - Convert user stories to JSON for autonomous execution

Skills are automatically invoked when you ask Claude to:

- "create a prd", "write prd for", "plan this feature", "spec out"
- "convert prd to ridl", "create ridl.md", "break prd into stories"
- "convert ridl.md to json", "create ridl.json", "ridl json"

## Acknowledgments

Inspired by [Ralph](https://github.com/snarktank/ralph), an autonomous coding agent loop by snarktank. RIDL builds on Ralph's approach to PRD-driven autonomous development and is deisgned to be driven by ralph loop interfaces such as [chief](https://github.com/MiniCodeMonkey/chief) or ridler.

