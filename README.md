# ridl-skills

RIDL = Ralph Iteration Definition List

A Claude Code plugin that provides a 3-step pipeline for generating and converting Product Requirements Documents (PRDs) into agent-sized iteration definitions for use in Ralph Loops.

An **iteration definition** is the unit of work in the RIDL system. Each one encompasses a user story, PRD references for traceability, and verifiable acceptance criteria — everything an autonomous agent needs to complete one iteration.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **prd** | `/ridl-skills:prd` | Generate a comprehensive PRD with requirements tables, technical architecture, user flows, and release milestones |
| **ridlmd** | `/ridl-skills:ridlmd` | Convert a PRD into `ridl.md` — agent-sized iteration definitions with user stories, PRD references, and verifiable acceptance criteria |
| **ridljson** | `/ridl-skills:ridljson` | Convert `ridl.md` into `ridl.json` — the JSON format consumed by a Ralph autonomous loop tool such as chief or ridler |

## Pipeline

The skills form a sequential pipeline. All files live in `ridl/`:

```
/ridl-skills:prd      →  ridl/prd.md      (comprehensive PRD)
/ridl-skills:ridlmd   →  ridl/ridl.md     (agent-sized iteration definitions)
/ridl-skills:ridljson  →  ridl/ridl.json   (JSON for autonomous ralph loop execution)
```

### Key Concepts

- **Iteration Definition (ID-###):** A self-contained unit of work completable in one agent iteration. Contains a user story, PRD references, and acceptance criteria.
- **Universal Context:** Cross-cutting constraints from the PRD (non-functional requirements, developer experience standards, architectural patterns) that apply to all iteration definitions. Agents must adhere to these regardless of which iteration definition they are implementing.
- **PRD References:** Each iteration definition traces back to specific requirement codes (e.g., TE-1, DI-3) from the source PRD for traceability.

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
- `/ridl-skills:ridlmd` - Convert PRDs to agent-sized iteration definitions
- `/ridl-skills:ridljson` - Convert iteration definitions to JSON for autonomous execution

Skills are automatically invoked when you ask Claude to:

- "create a prd", "write prd for", "plan this feature", "spec out"
- "convert prd to ridl", "create ridl.md", "break prd into iterations", "break prd into stories"
- "convert ridl.md to json", "create ridl.json", "ridl json"

## Acknowledgments

Inspired by [Ralph](https://github.com/snarktank/ralph), an autonomous coding agent loop by snarktank. RIDL builds on Ralph's approach to PRD-driven autonomous development and is designed to be driven by ralph loop interfaces such as [chief](https://github.com/MiniCodeMonkey/chief) or ridler.

