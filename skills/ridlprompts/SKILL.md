---
name: ridlprompts
version: 3.0.0
description: "Generate Liquid prompt template files for a RIDL harness (such as Ridler.app). This skill should be used when the user asks to 'generate ridl prompts', 'create ridl prompts', 'ridl prompts', 'generate loop prompts', 'create prompt templates', 'set up ridl harness', or 'initialize ridl prompts'. Produces the default set of Liquid templates that drive the RIDL autonomous coding agent loop."
user-invocable: true
---

# RIDL Prompt Template Generator

Generate the default set of Liquid prompt templates that a RIDL harness (such as Ridler.app) uses to drive its autonomous coding agent loop. These templates are the interface between the harness and the coding agent — they define what instructions the agent receives, what context it sees, and how it reports progress.

---

## The Job

1. Check if `ridl/ridl.json` exists (prompt the user to run `/ridl-skills:ridljson` first if not)
2. Check if `ridl/prompts/` already has template files
   - **No existing files:** Generate all 8 default templates
   - **Existing files with no modifications from defaults:** Overwrite with latest defaults
   - **Existing files with user modifications:** Merge updates and ask for feedback on conflicts (see Merge Behavior)
3. Save templates to `ridl/prompts/`

**Important:** Do NOT start implementing the project. Just create the prompt templates.

---

## Versioning

This skill uses semantic versioning (`major.minor.patch`) in the frontmatter `version` field. When Claude helps an author write or update this skill file, the version **must** be bumped before saving:

- **Patch** (e.g., 1.0.1 → 1.0.2): Small changes — typo fixes, wording tweaks, minor clarifications
- **Minor** (e.g., 1.0.2 → 1.1.0): Larger changes — adding/removing a section, changing the workflow, updating templates
- **Major** (e.g., 1.1.0 → 2.0.0): Very large changes in scope or features — fundamental restructuring, new capabilities, breaking changes to the output format

Always update the `version` field in the YAML frontmatter at the top of this file.

---

## Two-Phase Workflow

Each iteration definition goes through two phases across at least two separate agent invocations.

| Phase | Agent Invocation | Primary Template |
|-------|-----------------|-----------------|
| **Implementation** | Invocation 1 | `agent_instructions.liquid` |
| **Verification** | Invocation 2+ (fresh context) | `verification.liquid` |

The implementation agent writes failing tests, implements until green, updates runtime files as things come up, and commits. The verification agent starts fresh — no memory of the first invocation — independently confirms all criteria pass, fixes any issues, updates runtime files with final results, and signals completion.

### Verification Outcomes

The verification phase ends in one of three outcomes:

1. **Complete** — all criteria confirmed `"pass"`. The agent updates runtime files with final results and signals `<ridler-verification-complete/>`. The harness moves to the next iteration definition.

2. **Fix in place** — the verification agent finds issues where the fix is **clear and obvious** (e.g., a missing edge case, a small logic error, a formatting issue). If there is any ambiguity about the right fix, go back to implementation instead. Before fixing, it adds new acceptance criteria to `ridl/ridl.md` and `ridl/ridl.json` that describe the issue (set to `"status": "not_started"`). Then it writes a failing test (red), sets the new criteria to `"status": "fail"`, fixes until green, updates criteria to `"status": "pass"`, and commits. This ensures every fix is tracked with the same red/green discipline as the original implementation. Once all criteria are `"pass"`, it updates runtime files and signals `<ridler-verification-complete/>`.

3. **Back to implementation** — the verification agent finds issues it *cannot* fix in this context. It adds new acceptance criteria to `ridl/ridl.md` and `ridl/ridl.json` that describe the issue (set to `"status": "fail"`), documents what went wrong in `ridl/emergent.md` and/or `ridl/learnings.md`, and signals `<ridler-verification-failed/>`. The harness loops back to a **fresh implementation invocation** targeting just the failing criteria. After implementation addresses the failures, verification runs again.

This creates a natural loop: **Implementation → Verification → (if fail) → Implementation → Verification → ...** until all criteria reach `"pass"`. The per-criterion status gives the harness real-time visibility into which criteria are blocking completion.

### Harness Signals

Agents communicate completion state to the harness via XML tags at the end of their output:

| Signal | Meaning | Harness Action |
|--------|---------|---------------|
| `<ridler-implementation-complete/>` | Implementation finished, all criteria set to `"pass"` | Move to verification phase (fresh context window) |
| `<ridler-verification-complete/>` | Verification confirmed all criteria are `"pass"` | Move to next iteration definition |
| `<ridler-verification-failed/>` | Verification found issues the agent could not fix | Loop back to a fresh implementation invocation targeting failing criteria |
| `<ridler-blocked/>` | Cannot proceed — needs human input (unclear requirements, missing dependency, infrastructure issue) | Pause execution and surface the blocker to the human |

Every agent invocation must end with exactly one of these signals. If no signal is emitted, the harness should treat it as an unexpected crash and surface it to the human.

### Runtime Files

All phases can read from and append to `ridl/progress.md`, `ridl/learnings.md`, and `ridl/emergent.md` at any time:
- **progress.md** — append-only log of what was implemented and verified each iteration. Provides historical context for future agents.
- **learnings.md** — agent-facing: codebase patterns, tool quirks, implementation techniques, debugging tips. Helps future agents do the work better.
- **emergent.md** — human-facing: design decisions made without human input, requirement gaps, assumptions needing validation, scope questions, edge cases the PRD didn't anticipate. Items here should be backported to the PRD on regeneration.

---

## Merge Behavior

When `ridl/prompts/` already contains template files, the skill must decide whether to overwrite or merge:

### Step 1: Detect modifications

For each existing template file, compare it against the **previous default** (the default from the skill version that generated it). A template is considered **modified** if the user has made changes beyond what the defaults provide.

Since there is no reliable way to track which skill version generated a file, use a pragmatic approach:

- Read each existing template file
- Compare it to the current default template defined in this skill
- If the content is **identical** to the current default: overwrite silently (no user changes to preserve)
- If the content **differs** from the current default: the file has been customized

### Step 2: Handle customized templates

For each customized template:

1. Show the user what changed between the current default and the new default (the update this skill wants to apply)
2. Show what customizations the user has made
3. Attempt to merge — apply the default updates while preserving user customizations
4. Present the merged result and ask for confirmation
5. If the merge is ambiguous or conflicts exist, show both versions and ask the user to choose

### Step 3: Confirm and save

After all templates are reviewed, save the final versions to `ridl/prompts/`.

---

## Template Files

The skill generates 8 Liquid template files. These are the canonical defaults — a RIDL harness reads these at runtime and fills in the Liquid variables with data from `ridl/ridl.json` and runtime state.

### 1. `agent_instructions.liquid`

The implementation phase prompt. This is the primary instruction set given to the coding agent at the start of each iteration. It tells the agent what to do: read the PRD, pick the next iteration definition, write failing tests, implement, run checks, and commit.

~~~liquid
# Implementation Agent Instructions

**Generated by:** ridl-skills:ridlprompts v{{ skill_version }}

You are an autonomous coding agent working on a software project. This is the **implementation phase** (phase 1 of 2).

## Your Task

1. Read the PRD at `ridl/ridl.json`
2. Read `ridl/emergent.md`, `ridl/progress.md`, and `ridl/learnings.md` if they exist (check Codebase Patterns section first)
3. **At any time** during this phase, append to these files as things come up — do not wait until the end:
   - `ridl/learnings.md` — for **agent-facing** knowledge: codebase patterns, tool quirks, implementation techniques that worked or didn't, debugging tips, environment gotchas. Things that help future agents do the work better.
   - `ridl/emergent.md` — for **human-facing** items that need to flow back to the PRD: design decisions made without human input, gaps or ambiguities in the requirements, assumptions that should be validated, scope questions, edge cases the PRD didn't anticipate.
4. Pick the **highest priority** iteration definition where any acceptance criterion has `"status": "not_started"` or `"status": "fail"` -- After determining which iteration definition to work on, output its id: <ridler-status>{{ iteration.id }}</ridler-status>
5. **Write failing tests first** — for each acceptance criterion, write a test that verifies it. Tests must fail before implementation (red).
6. **Update `ridl/ridl.json`** — set each criterion you wrote tests for to `"status": "fail"`. This confirms the red phase.
7. **Implement** until all tests pass (green)
8. **Update `ridl/ridl.json`** — set each now-passing criterion to `"status": "pass"`.
9. **Every time you run tests**, update ridl/ridl.json to reflect the current state. Criterion statuses should always match reality — `"not_started"` if not yet attempted, `"fail"` if the test fails, `"pass"` if it passes, `"error"` if something unexpected prevents evaluation.
10. Run quality checks (typecheck, lint, test, format — use whatever your project requires)
    - **If the format check fails**, run the format fix command first (e.g., `ruff format .` instead of `ruff format --check .`). Most formatting issues are mechanical — let the formatter fix them automatically. Only investigate manually if the fix command itself fails or produces unexpected results.
11. If checks pass, commit ALL changes with message: `feature: [{{ iteration.id }}] implementation - {{ iteration.title }}`

## Red/Green Discipline

The acceptance criteria in ridl/ridl.json track status in real time:
- `"not_started"` — you haven't touched this criterion yet
- `"fail"` — you wrote a test and it fails (this is expected and correct during the red phase)
- `"pass"` — implementation makes the test pass
- `"error"` — something unexpected went wrong

Update criterion statuses as you work so the harness can observe progress.

## Important

- Do NOT skip the red phase. Every test must demonstrably fail before you write the implementation.
- Do NOT move on to verification. Just implement and commit.
- Keep changes focused on this single iteration definition.
- If you are re-implementing after a failed verification, focus on the criteria with `"status": "fail"` — read `ridl/emergent.md` and `ridl/learnings.md` for context on what went wrong.

## Completion

When implementation is done, signal with exactly one of:
- `<ridler-implementation-complete/>` — implementation finished, ready for verification
- `<ridler-blocked/>` — cannot proceed, needs human input (document the blocker in `ridl/emergent.md`)
~~~

### 2. `verification.liquid`

The verification phase prompt. A fresh agent invocation reviews the implementation independently — no memory of the first invocation. It runs checks, looks for issues, confirms all acceptance criteria genuinely pass, updates runtime files with final results, and signals completion. This phase may require multiple invocations if issues are complex.

~~~liquid
# Verification Agent Instructions

**Generated by:** ridl-skills:ridlprompts v{{ skill_version }}

You are an autonomous verification agent reviewing work done by a separate implementation agent. You are in a **fresh context window** with no memory of the implementation. This is the **verification phase** (phase 2 of 2).

## Verification

1. Read `ridl/ridl.json` and identify the iteration definition that was just implemented (look for the one with a mix of `"pass"` and potentially some non-pass criteria, or the most recently modified one at the highest priority)
2. Read `ridl/emergent.md`, `ridl/progress.md`, and `ridl/learnings.md` for context
3. **At any time** during this phase, append to these files as things come up — do not wait until the end:
   - `ridl/learnings.md` — for **agent-facing** knowledge: codebase patterns, tool quirks, implementation techniques that worked or didn't, debugging tips, environment gotchas. Things that help future agents do the work better.
   - `ridl/emergent.md` — for **human-facing** items that need to flow back to the PRD: design decisions made without human input, gaps or ambiguities in the requirements, assumptions that should be validated, scope questions, edge cases the PRD didn't anticipate.
4. Read the implementation commit to understand what changed
5. **Independently verify each acceptance criterion:**
   - Run the test suite — do all tests actually pass?
   - Run typecheck, lint, format checks — all clean?
   - For UI criteria: verify in browser using dev-browser skill
   - Read the code — does it actually satisfy each criterion, or just superficially pass tests?
6. **Look for issues the implementation agent may have missed:**
   - Edge cases not covered by tests
   - Regressions in existing functionality
   - Code quality concerns (security, performance, maintainability)
   - Acceptance criteria that technically pass but miss the spirit of the requirement
7. If you find issues where the fix is **clear and obvious** (if there is any ambiguity, go to "If Verification Fails" instead):
   a. Add new acceptance criteria describing each issue to `ridl/ridl.md` and `ridl/ridl.json` (set to `"status": "not_started"`)
   b. Write a failing test for each new criterion (red) — set to `"status": "fail"`
   c. Fix until tests pass (green) — set to `"status": "pass"`
   d. Commit: `fix: [{{ iteration.id }}] verification - {{ iteration.title }}`
8. If everything checks out, commit any minor improvements with: `verify: [{{ iteration.id }}] verification - {{ iteration.title }}`
9. Update `ridl/ridl.json` — confirm or correct criterion statuses based on your independent assessment

### If Verification Fails

If you find issues you **cannot** fix in this context:

1. Set relevant criteria to `"status": "fail"` and/or add new acceptance criteria describing each issue to `ridl/ridl.md` and `ridl/ridl.json` (set to `"status": "fail"`)
2. Document what went wrong and why in `ridl/emergent.md`
3. Document any useful context for the next attempt in `ridl/learnings.md`
4. Signal: `<ridler-verification-failed/>` — the harness will loop back to a fresh implementation invocation targeting the failing criteria

### If Blocked

If you **cannot proceed** for reasons outside your control (missing dependency, unclear requirements, needs human input, infrastructure issue):

1. Set the relevant criteria to `"status": "error"` in ridl/ridl.json
2. Document the blocker in `ridl/emergent.md`
3. Signal: `<ridler-blocked/>` — the harness will pause and surface this to the human

## Before Signaling Completion

Once all criteria are confirmed `"pass"`, update runtime files with final results:

1. Append to `ridl/progress.md`:
   ```
   ## [Date/Time] - [{{ iteration.id }}] {{ iteration.title }}
   - **Phase:** verification
   - What was implemented
   - Files changed
   - Verification results (what the verification agent confirmed or fixed)
   - How we proved this was successfully implemented (tests passed, manual verification, etc.)
   ---
   ```
2. Append to `ridl/learnings.md` if any patterns or gotchas were discovered (prefix each item with a category: `pattern:`, `gotcha:`, `technique:`, `tool:`, `debug:`, `context:`, or `other:`):
   ```
   ## [Date/Time] - [{{ iteration.id }}] {{ iteration.title }}
   - **Phase:** verification
   - pattern: [description]
   - gotcha: [description]
   - technique: [description]
   ---
   ```
3. Append to `ridl/emergent.md` if any design decisions, PRD ambiguities, assumptions, or edge cases came up. Assign the next available `EM-N` ID:
   ```
   ## EM-N: [Short title]
   - **Source:** [{{ iteration.id }}] {{ iteration.title }}
   - **Date:** [Date/Time]
   - **Phase:** verification
   - **Category:** [design decision | requirement gap | assumption | scope question | edge case]
   - **Outcome:** pending
   - What came up
   - What was decided or assumed (if anything)
   - Why this needs human review
   ---
   ```
4. Confirm all criteria in `ridl/ridl.json` are `"status": "pass"`
5. Signal completion: <ridler-verification-complete/>

## Verification Mindset

You are a fresh pair of eyes. The implementation agent may have:
- Written tests that pass but don't actually verify the criterion
- Missed edge cases that the acceptance criteria imply but don't spell out
- Made assumptions that should be flagged in `ridl/emergent.md`
- Introduced subtle regressions

Be thorough but constructive. The goal is confidence that the iteration definition is truly complete, not finding fault for its own sake.
~~~

### 3. `iteration_context.liquid`

Injects the target iteration definition's details into the agent's context. Includes the title, description, acceptance criteria with current statuses, PRD references, and any universal context that applies across all iterations.

~~~liquid
## Target Iteration Definition

- **Generated by:** ridl-skills:ridlprompts v{{ skill_version }}
- **ID:** {{ iteration.id }}
- **Title:** {{ iteration.title }}
- **Priority:** {{ iteration.priority }}
- **Description:** {{ iteration.description }}

### Acceptance Criteria

{% for ac in iteration.acceptance_criteria %}
- [{{ ac.status }}] {{ ac.criterion }}
{% endfor %}

{% if iteration.prd_references %}
### PRD References

{% for ref in iteration.prd_references %}
- {{ ref }}
{% endfor %}
{% endif %}

{% if project.has_universal_context %}
## Universal Context

{% if project.non_functional_requirements %}
### Non-Functional Requirements

{% for item in project.non_functional_requirements %}
- {{ item }}
{% endfor %}
{% endif %}

{% if project.developer_experience %}
### Developer Experience

{% for item in project.developer_experience %}
- {{ item }}
{% endfor %}
{% endif %}

{% if project.technical_architecture %}
### Technical Architecture

{% for item in project.technical_architecture %}
- {{ item }}
{% endfor %}
{% endif %}

{% if project.testing_and_verification %}
### Testing & Verification

{% for item in project.testing_and_verification %}
- {{ item }}
{% endfor %}
{% endif %}

{% endif %}
~~~

### 4. `progress_format.liquid`

Format reference for progress logging. Any agent appends to `ridl/progress.md` using this format, tagging each entry with the current phase.

~~~liquid
# Progress Report Format

**Generated by:** ridl-skills:ridlprompts v{{ skill_version }}

This file tracks what was implemented in each RIDL iteration and how it was verified. Append-only — never replace existing entries.

---

APPEND to ridl/progress.md (never replace, always append):

```
## [Date/Time] - [{{ iteration.id }}] {{ iteration.title }}
- **Phase:** [implementation | verification]
- What was implemented
- Files changed
- Verification results (what the verification agent confirmed or fixed)
- How we proved this was successfully implemented (tests passed, manual verification, etc.)
---
```

## Quality Requirements

- ALL commits must pass your project's quality checks (typecheck, lint, test)
- Do NOT commit broken code
- Keep changes focused and minimal
- Follow existing code patterns

{% if progress_content %}
## Previous Progress

{{ progress_content }}
{% endif %}

~~~

### 5. `learnings_format.liquid`

Format reference for capturing learnings and institutional knowledge. Any agent appends to `ridl/learnings.md` using this format, tagging each entry with the current phase.

~~~liquid
# Learnings Format

**Generated by:** ridl-skills:ridlprompts v{{ skill_version }}

This file captures patterns, gotchas, and useful context discovered during RIDL iterations. Future agents read this to avoid repeating mistakes and to build on prior knowledge. Append-only — never replace existing entries.

---

APPEND to ridl/learnings.md (never replace, always append):

Prefix each line item with a category tag:
- `pattern:` — codebase conventions, naming patterns, file organization
- `gotcha:` — surprising behavior, pitfalls, counterintuitive things
- `technique:` — implementation approaches that worked (or didn't)
- `tool:` — CLI quirks, tool flags, configuration workarounds
- `debug:` — how to diagnose specific issues, useful log lines
- `context:` — background knowledge useful for future iterations
- `other:` — catch-all

```
## [Date/Time] - [{{ iteration.id }}] {{ iteration.title }}
- **Phase:** [implementation | verification]
- pattern: [description]
- gotcha: [description]
- technique: [description]
---
```

{% if learnings_content %}
## Previous Learnings

{{ learnings_content }}
{% endif %}
~~~

### 6. `emergent_format.liquid`

Format reference for capturing emergent items that need human attention. Any agent can append to `ridl/emergent.md` at any time during implementation or verification.

~~~liquid
# Emergent Items Format

**Generated by:** ridl-skills:ridlprompts v{{ skill_version }}

This file captures items that emerged during RIDL iterations and need human attention — design decisions made without human input, requirement gaps, assumptions needing validation, scope questions, and edge cases the PRD didn't anticipate. Items here should be backported to the PRD on regeneration. Append-only — never replace existing entries.

---

APPEND to ridl/emergent.md (never replace, always append):

Each item gets a unique ID: `EM-N` (incrementing from the last ID in the file, or EM-1 if the file is empty). This ID lets the PRD reference specific emergent items after triage.

```
## EM-N: [Short title]
- **Source:** [{{ iteration.id }}] {{ iteration.title }}
- **Date:** [Date/Time]
- **Phase:** [implementation | verification]
- **Category:** [design decision | requirement gap | assumption | scope question | edge case]
- **Outcome:** pending
- What came up
- What was decided or assumed (if anything)
- Why this needs human review
---
```

Outcome values: `pending` (default at creation), `approved`, `approved with changes`, `rejected`, `deferred`. The outcome is updated during PRD triage (see `/ridl-skills:prd`).

{% if emergent_content %}
## Previous Emergent Items

{{ emergent_content }}
{% endif %}
~~~

### 7. `edit_file.liquid`

A utility prompt for editing existing PRD or RIDL files. A RIDL harness uses this when the user wants to modify an existing file through the agent.

~~~liquid
I want to edit the PRD file at: {{ file_path }}

Please read the file and help me modify it. Show me the current contents first.
~~~

### 8. `create_file.liquid`

A utility prompt for creating missing RIDL pipeline files. A RIDL harness uses this when a required file doesn't exist yet and needs to be bootstrapped from existing artifacts.

~~~liquid
{% if file_name == "ridl.md" %}
The file {{ file_path }} does not exist yet.
Please create ridl.md from the existing prd.md in the same directory.
Read prd.md first, then create ridl.md with iteration definitions and acceptance criteria.

{% elsif file_name == "ridl.json" %}
The file {{ file_path }} does not exist yet.
Please create ridl.json from the existing ridl.md or prd.md in the same directory.
Read the existing files first, then create ridl.json in the proper format.

{% elsif file_name == "emergent.md" %}
The file {{ file_path }} does not exist yet.
Please create emergent.md with an empty template for capturing emergent design decisions, PRD ambiguities, assumptions, and edge cases discovered during RIDL iterations.

{% else %}
The file {{ file_path }} does not exist yet.
Please create it with appropriate initial content.

{% endif %}
~~~

---

## Output

- **Format:** Liquid template files (`.liquid`)
- **Location:** `ridl/prompts/` directory at the project root
- **Files:**
  - `agent_instructions.liquid`
  - `verification.liquid`
  - `iteration_context.liquid`
  - `progress_format.liquid`
  - `learnings_format.liquid`
  - `emergent_format.liquid`
  - `edit_file.liquid`
  - `create_file.liquid`

Create the `ridl/prompts/` directory if it does not exist.

---

## Pipeline Context

This skill is step 4 in the RIDL pipeline. All files live in `ridl/`:

```
/ridl-skills:prd         →  ridl/prd.md             (comprehensive PRD)
/ridl-skills:ridlmd      →  ridl/ridl.md            (agent-sized iteration definitions)
/ridl-skills:ridljson    →  ridl/ridl.json          (JSON for autonomous loop)
/ridl-skills:ridlprompts →  ridl/prompts/*.liquid   (harness prompt templates)  ← you are here
```

---

## Checklist

Before saving the templates:

- [ ] Verified `ridl/ridl.json` exists (or prompted user to create it)
- [ ] Checked for existing templates in `ridl/prompts/`
- [ ] Unmodified existing templates overwritten with latest defaults
- [ ] Modified existing templates merged with user review
- [ ] All 8 template files generated:
  - [ ] `agent_instructions.liquid`
  - [ ] `verification.liquid`
  - [ ] `iteration_context.liquid`
  - [ ] `progress_format.liquid`
  - [ ] `learnings_format.liquid`
  - [ ] `emergent_format.liquid`
  - [ ] `edit_file.liquid`
  - [ ] `create_file.liquid`
- [ ] Templates reference per-criterion status objects (not single `passes` boolean)
- [ ] Templates reference `"title"` and `"description"` keys (not legacy `"userStoryTitle"` / `"userStoryDescription"`)
- [ ] Two-phase workflow is represented (implementation → verification)
- [ ] Verification template includes failure strategy (add new criteria to `ridl/ridl.md` and `ridl/ridl.json`, set criteria to fail, document in ridl/emergent.md, harness loops back)
- [ ] All relevant templates reference `ridl/progress.md`, `ridl/learnings.md`, and `ridl/emergent.md` for reading and appending
- [ ] Templates saved to `ridl/prompts/`
