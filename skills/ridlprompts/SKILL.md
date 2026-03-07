---
name: ridlprompts
version: 2.2.0
description: "Generate Liquid prompt template files for a RIDL harness (such as Ridler.app). This skill should be used when the user asks to 'generate ridl prompts', 'create ridl prompts', 'ridl prompts', 'generate loop prompts', 'create prompt templates', 'set up ridl harness', or 'initialize ridl prompts'. Produces the default set of Liquid templates that drive the RIDL autonomous coding agent loop."
user-invocable: true
---

# RIDL Prompt Template Generator

Generate the default set of Liquid prompt templates that a RIDL harness (such as Ridler.app) uses to drive its autonomous coding agent loop. These templates are the interface between the harness and the coding agent — they define what instructions the agent receives, what context it sees, and how it reports progress.

---

## The Job

1. Check if `ridl/ridl.json` exists (prompt the user to run `/ridl-skills:ridljson` first if not)
2. Check if `ridl/prompts/` already has template files
   - **No existing files:** Generate all 5 default templates
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

The skill generates 5 Liquid template files. These are the canonical defaults — a RIDL harness reads these at runtime and fills in the Liquid variables with data from `ridl.json` and runtime state.

### 1. `agent_instructions.liquid`

The main loop prompt. This is the primary instruction set given to the coding agent at the start of each iteration. It tells the agent what to do: read the PRD, pick the next story, implement it, run checks, commit, and update status.

~~~liquid
# Chief Agent Instructions

You are an autonomous coding agent working on a software project.

## Your Task

1. Read the PRD at `ridl/ridl.json`
2. Read `progress.md` if it exists (check Codebase Patterns section first)
3. Pick the **highest priority** iteration definition where `passes: false` -- After determining which story to work on, output exact story id, e.g.: <ralph-status>{{ story.id }}</ralph-status>
4. Implement that single iteration definition
5. Run quality checks (e.g., typecheck, lint, test - use whatever your project requires)
6. If checks pass, commit ALL changes with message: `feature: [{{ story.id }}] - {{ story.title }}`
7. Append your progress to `progress.md`
8. **LAST STEP — do this after everything else is done:** Update `ridl/ridl.json` to set `"passes": true` for the completed iteration definition. This MUST be the final action you take, outside of any cleanup tasks or final logging for debug or non-user facing purposes.
~~~

### 2. `story_context.liquid`

Injects the target story's details into the agent's context. Includes the user story, acceptance criteria, PRD references, and any universal context (non-functional requirements, developer experience standards, technical architecture constraints) that apply across all iterations.

~~~liquid
## Target Story

- **ID:** {{ story.id }}
- **Title:** {{ story.title }}
- **Priority:** {{ story.priority }}
- **Description:** {{ story.description }}

### Acceptance Criteria

{% for criterion in story.acceptance_criteria %}
- {{ criterion }}
{% endfor %}

{% if story.prd_references %}
### PRD References

{% for ref in story.prd_references %}
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

{% endif %}
~~~

### 3. `progress_report.liquid`

Defines the format for progress logging and the stop condition. The agent appends to `progress.md` after each iteration and signals completion with the `<ridler-complete/>` tag.

~~~liquid
## Progress Report Format

APPEND to progress.md (never replace, always append):

```
## [Date/Time] - [{{ story.id }}]
- What was implemented
- Files changed
- **Learnings for future iterations:**
  - Patterns discovered
  - Gotchas encountered
  - Useful context
---
```

## Quality Requirements

- ALL commits must pass your project's quality checks (typecheck, lint, test)
- Do NOT commit broken code
- Keep changes focused and minimal
- Follow existing code patterns

## Stop Condition

After completing the user story, reply with:
<ridler-complete/>

{% if progress_content %}
## Previous Progress

{{ progress_content }}
{% endif %}
~~~

### 4. `edit_file.liquid`

A utility prompt for editing existing PRD or RIDL files. A RIDL harness uses this when the user wants to modify an existing file through the agent.

~~~liquid
I want to edit the PRD file at: {{ file_path }}

Please read the file and help me modify it. Show me the current contents first.
~~~

### 5. `create_file.liquid`

A utility prompt for creating missing RIDL pipeline files. A RIDL harness uses this when a required file doesn't exist yet and needs to be bootstrapped from existing artifacts.

~~~liquid
{% if file_name == "ridl.md" %}
The file {{ file_path }} does not exist yet.
Please create ridl.md from the existing prd.md in the same directory.
Read prd.md first, then create ridl.md with user stories and acceptance criteria.

{% elsif file_name == "ridl.json" %}
The file {{ file_path }} does not exist yet.
Please create ridl.json from the existing ridl.md or prd.md in the same directory.
Read the existing files first, then create ridl.json in the proper format.

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
  - `story_context.liquid`
  - `progress_report.liquid`
  - `edit_file.liquid`
  - `create_file.liquid`

Create the `ridl/prompts/` directory if it does not exist.

---

## Pipeline Context

This skill is step 4 in the RIDL pipeline. All files live in `ridl/`:

```
/ridl-skills:prd        →  ridl/prd.md        (comprehensive PRD)
/ridl-skills:ridlmd     →  ridl/ridl.md       (agent-sized iteration definitions)
/ridl-skills:ridljson   →  ridl/ridl.json     (JSON for autonomous loop)
/ridl-skills:ridlprompts →  ridl/prompts/*.liquid  (harness prompt templates)  ← you are here
```

---

## Checklist

Before saving the templates:

- [ ] Verified `ridl/ridl.json` exists (or prompted user to create it)
- [ ] Checked for existing templates in `ridl/prompts/`
- [ ] Unmodified existing templates overwritten with latest defaults
- [ ] Modified existing templates merged with user review
- [ ] All 5 template files generated:
  - [ ] `agent_instructions.liquid`
  - [ ] `story_context.liquid`
  - [ ] `progress_report.liquid`
  - [ ] `edit_file.liquid`
  - [ ] `create_file.liquid`
- [ ] Templates saved to `ridl/prompts/`
