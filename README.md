# ridl-skills

RIDL = Ralph Iteration Definition List

RIDL is a very opiniated implementation of the high level Ralph Loop idea.

Inspired by Ralph Loops, this is a Claude Code plugin that provides a 4-step pipeline for generating and converting Product Requirements Documents (PRDs) into agent-sized iteration definitions and prompt templates for use by a RIDL harness (such as Ridler.app).  

An **iteration definition** is the unit of work in the RIDL system. Each one encompasses a user story, PRD references for traceability, and verifiable acceptance criteria — everything an autonomous agent needs to complete one iteration.

## Skills

| Skill | Command | Description |
|-------|---------|-------------|
| **prd** | `/ridl-skills:prd` | Generate a comprehensive PRD with requirements tables, technical architecture, user flows, and release milestones |
| **ridlmd** | `/ridl-skills:ridlmd` | Convert a PRD into `ridl.md` — agent-sized iteration definitions with user stories, PRD references, and verifiable acceptance criteria |
| **ridljson** | `/ridl-skills:ridljson` | Convert `ridl.md` into `ridl.json` — the JSON format consumed by a RIDL harness |
| **ridlprompts** | `/ridl-skills:ridlprompts` | Generate default Liquid prompt templates into `ridl/prompts/` for a RIDL harness |

## Pipeline

The skills form a sequential pipeline. All files live in `ridl/`:

```
/ridl-skills:prd        →  ridl/prd.md              (comprehensive PRD)
/ridl-skills:ridlmd     →  ridl/ridl.md             (agent-sized iteration definitions)
/ridl-skills:ridljson   →  ridl/ridl.json           (JSON for autonomous ralph loop-style execution)
/ridl-skills:ridlprompts →  ridl/prompts/*.liquid   (prompt templates for a RIDL harness)
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
- `/ridl-skills:ridlprompts` - Generate prompt templates for a RIDL harness

Skills are automatically invoked when you ask Claude to:

- "create a prd", "write prd for", "plan this feature", "spec out"
- "convert prd to ridl", "create ridl.md", "break prd into iterations", "break prd into stories"
- "convert ridl.md to json", "create ridl.json", "ridl json"
- "generate ridl prompts", "create ridl prompts", "ridl prompts", "create prompt templates"

## Acknowledgments

Inspired by [Ralph skills](https://github.com/snarktank/ralph), an autonomous coding agent loop by snarktank, and [chief](https://github.com/MiniCodeMonkey/chief), a helper companion harness. 

RIDL builds on Ralph's approach to PRD-driven autonomous development. [Ridler.app](https://ridler.app) is the RIDL-native harness, filling the same role for RIDL that chief fills for snarktank's ralph skills.

