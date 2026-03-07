# ridl-skills

RIDL = Ralph Iteration Definition List

RIDL is a very opinionated implementation of the high level Ralph Loop idea.

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
/ridl-skills:prd         →  ridl/prd.md             (comprehensive PRD)
/ridl-skills:ridlmd      →  ridl/ridl.md            (agent-sized iteration definitions)
/ridl-skills:ridljson    →  ridl/ridl.json          (JSON for autonomous loop)
/ridl-skills:ridlprompts →  ridl/prompts/*.liquid   (harness prompt templates)
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
- `/ridl-skills:ridlmd` - Convert PRDs to agent-sized iteration definitions
- `/ridl-skills:ridljson` - Convert iteration definitions to JSON for autonomous execution
- `/ridl-skills:ridlprompts` - Generate prompt templates for a RIDL harness

Skills are automatically invoked when you ask Claude to:

- "create a prd", "write prd for", "plan this feature", "spec out"
- "convert prd to ridl", "create ridl.md", "break prd into iterations", "break prd into stories"
- "convert ridl.md to json", "create ridl.json", "ridl json"
- "generate ridl prompts", "create ridl prompts", "ridl prompts", "create prompt templates"

## How to Use

### 1. Generate Pipeline Files

Run the skills in order to set up your project for autonomous execution:

1. **Create a PRD** — `/ridl-skills:prd` — describe your feature and answer clarifying questions
2. **Generate iteration definitions** — `/ridl-skills:ridlmd` — breaks the PRD into agent-sized units of work
3. **Convert to JSON** — `/ridl-skills:ridljson` — produces the `ridl.json` consumed by the harness
4. **Generate prompt templates** — `/ridl-skills:ridlprompts` — creates the Liquid templates that drive the agent loop

### 2. Start the Loop

Launch your RIDL harness (e.g., [Ridler.app](https://github.com/amattn/ridler)). The harness picks up `ridl/ridl.json` and the templates in `ridl/prompts/`, then drives agents through iteration definitions sequentially using the two-phase workflow (implementation → verification).

### 3. Review and Steer

When you have time — or when the harness completes a batch of iterations or signals a blocker — pause the loop and review what the agents produced:

1. **Read `ridl/emergent.md`** — design decisions made without your input, requirement gaps, assumptions, scope questions. These are items the agents flagged for human review.
2. **Read `ridl/progress.md`** — what was implemented and verified in each iteration.
3. **Read `ridl/learnings.md`** — patterns, gotchas, and techniques the agents discovered.

### 4. Update and Continue

Based on your review, make any needed adjustments before resuming:

1. **Update the PRD** — re-run `/ridl-skills:prd` to triage emergent items, answer open questions, and refine requirements. Approved items get incorporated; rejected items go to Resolved Questions.
2. **Update iteration definitions** — re-run `/ridl-skills:ridlmd` to append new iteration definitions or edit unfinished ones based on PRD changes.
3. **Update JSON** — re-run `/ridl-skills:ridljson` to sync `ridl.json` with the updated `ridl.md`. Frozen (completed) iteration definitions are preserved automatically.
4. **Resume the loop** — the harness picks up where it left off, targeting the next non-frozen iteration definition.

Repeat steps 2–4 as needed. The loop is designed to run autonomously, with periodic human steering to keep the project on track.

## Key Concepts

### Iteration Definitions

- **Iteration Definition (ID-###):** A self-contained unit of work completable in one agent context window. Contains a user story, PRD references, and acceptance criteria.
- **Universal Context:** Cross-cutting constraints from the PRD (non-functional requirements, developer experience standards, architectural patterns) that apply to all iteration definitions.
- **PRD References:** Each iteration definition traces back to specific requirement codes (e.g., TE-1, DI-3) from the source PRD for traceability.
- **Frozen Definitions:** When all acceptance criteria in an iteration definition reach `"pass"`, the definition is frozen and must not be modified. New work is appended as new iteration definitions.

### Per-Criterion Status Tracking

Each acceptance criterion in `ridl.json` is a structured object with a `"criterion"` string and a `"status"` enum:

| Status | Meaning |
|--------|---------|
| `"not_started"` | Criterion has not been attempted yet |
| `"fail"` | Criterion was tested and is currently failing (the "red" in red/green) |
| `"pass"` | Criterion has been verified and is passing |
| `"error"` | Unexpected error — criterion could not be evaluated |

This replaces the legacy single `"passes"` boolean, giving fine-grained visibility into iteration progress.

### Two-Phase Workflow

Each iteration definition goes through two phases across at least two separate agent invocations:

1. **Implementation** — The agent writes failing tests first, then implements until green (red/green discipline). Updates runtime files (progress.md, learnings.md, emergent.md) as things come up.
2. **Verification** — Independent check in a fresh context window. A second agent invocation reviews the work, confirms all criteria pass, and updates runtime files with final results.

Verification ends in one of three outcomes:

1. **Complete** — All criteria confirmed `"pass"`. Runtime files updated, agent signals `<ridler-verification-complete/>`.
2. **Fix in place** — Issues where the fix is clear and obvious. The verification agent adds new acceptance criteria to `ridl/ridl.md` and `ridl/ridl.json`, then fixes with red/green discipline.
3. **Back to implementation** — Issues the verification agent cannot fix. It adds new acceptance criteria set to `"fail"` and signals `<ridler-verification-failed/>`, looping back to a fresh implementation invocation.

### Runtime Files

The RIDL harness maintains three runtime files alongside the iteration data:

- **`ridl/progress.md`** — Append-only log of what was implemented and verified each iteration
- **`ridl/learnings.md`** — Agent-facing: codebase patterns, tool quirks, implementation techniques, debugging tips. Helps future agents do the work better.
- **`ridl/emergent.md`** — Human-facing: design decisions made without human input, requirement gaps, assumptions needing validation, scope questions, edge cases the PRD didn't anticipate. Items here should be backported to the PRD on regeneration.

### Harness Signals

Agents communicate completion state to the harness via XML signals:

- `<ridler-implementation-complete/>` — Implementation finished, move to verification
- `<ridler-verification-complete/>` — Verification confirmed all criteria pass, iteration done
- `<ridler-verification-failed/>` — Verification found issues, needs retry
- `<ridler-blocked/>` — Cannot proceed, needs human intervention

### Prompt Templates

The `ridlprompts` skill generates 8 Liquid templates that define the interface between the harness and the coding agent:

| Template | Purpose |
|----------|---------|
| `agent_instructions.liquid` | Core agent instructions for implementation |
| `verification.liquid` | Verification phase instructions |
| `iteration_context.liquid` | Current iteration definition context |
| `progress_format.liquid` | Format for progress.md updates |
| `learnings_format.liquid` | Format for learnings.md entries |
| `emergent_format.liquid` | Format for emergent.md items (with Outcome tracking) |
| `edit_file.liquid` | File editing conventions |
| `create_file.liquid` | File creation conventions |

### Emergent Items and Triage

When agents discover requirements not covered by the current PRD during implementation, they log them to `ridl/emergent.md` with EM-N identifiers. Each item has a default `Outcome: pending` that gets updated during the next PRD triage:

- **Approved** / **Approved with changes** — Incorporated into the PRD with EM-N reference
- **Rejected** — Noted in PRD Resolved Questions with EM-N reference
- **Deferred** — Added to PRD Open Questions, stays in emergent.md for next triage

## Implementing a RIDL Harness

A RIDL harness orchestrates the autonomous execution loop. It consumes `ridl/ridl.json` and the Liquid templates in `ridl/prompts/` to drive coding agents through iteration definitions sequentially. Here's what a minimal harness implementation needs to do:

1. **Load `ridl/ridl.json`** and identify the next non-frozen iteration definition (the first where any acceptance criterion is not `"pass"`)
2. **Validate and Render Liquid templates** — fill in `iteration_context.liquid` with the current iteration definition data, combine with `agent_instructions.liquid` (implementation) or `verification.liquid` (verification phase), and include `edit_file.liquid` / `create_file.liquid` as tool-use conventions
3. **Invoke a coding agent** with the rendered prompt. The agent works through acceptance criteria using red/green discipline
4. **Parse the harness signal** emitted at the end of the agent's output:
   - `<ridler-implementation-complete/>` → move to verification phase (fresh context window)
   - `<ridler-verification-complete/>` → mark iteration done, advance to next iteration definition
   - `<ridler-verification-failed/>` → loop back to a fresh implementation invocation targeting failing criteria
   - `<ridler-blocked/>` → pause and surface the blocker to the human
   - No signal → treat as unexpected crash
5. **Update `ridl/ridl.json`** — the agent updates per-criterion statuses in real time; the harness can observe progress between `"not_started"` → `"fail"` → `"pass"`
6. **Manage runtime files** — ensure `ridl/progress.md`, `ridl/learnings.md`, and `ridl/emergent.md` are accessible to agents for reading and appending

The two-phase workflow (implementation → verification) and the three verification outcomes (complete, fix in place, back to implementation) are defined in the prompt templates. The harness's job is to render those templates, invoke agents, and route based on signals.

## Acknowledgments

Inspired by [Ralph skills](https://github.com/snarktank/ralph), an autonomous coding agent loop by snarktank, and [chief](https://github.com/MiniCodeMonkey/chief), a helper companion harness.

RIDL builds on Ralph's approach to PRD-driven autonomous development. [Ridler.app](https://github.com/amattn/ridler) is a Mac OS X-native RIDL harness, filling the same role for RIDL that chief fills for snarktank's ralph skills.
