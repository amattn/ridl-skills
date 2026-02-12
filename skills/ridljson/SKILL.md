---
name: ridljson
version: 2.0.0
description: "Convert ridl.md to ridl.json format for the RIDL autonomous agent system. This skill should be used when the user asks to 'convert ridl.md', 'turn this into ridl json', 'create ridl.json', 'ridl json', 'convert ridl.md to json', or 'make ridl iterations from ridl.md'."
user-invocable: true
---

# RIDL PRD Converter

Convert existing ridl.md to the ridl.json format that RIDL uses for autonomous execution.

---

## The Job

Read `ridl.md` from `ridl/` (or a path specified by the user) and convert it to `ridl.json` in the same directory.

If no `ridl/ridl.md` is found, prompt the user to either provide a path or run `/ridl-skills:ridlmd` first to generate one.

---

## Versioning

**Skill file versioning:** This skill uses semantic versioning (`major.minor.patch`) in the frontmatter `version` field. When Claude helps an author write or update this skill file, the version **must** be bumped before saving:

- **Patch** (e.g., 1.0.1 → 1.0.2): Small changes — typo fixes, wording tweaks, minor clarifications
- **Minor** (e.g., 1.0.2 → 1.1.0): Larger changes — adding/removing a section, changing the workflow, updating templates
- **Major** (e.g., 1.1.0 → 2.0.0): Very large changes in scope or features — fundamental restructuring, new capabilities, breaking changes to the output format

Always update the `version` field in the YAML frontmatter at the top of this file.

**Output versioning:** The generated `ridl.json` **must** inherit the version from the source `ridl.md`. Read the `**Version:**` field from the ridl.md header and include it as the `"version"` field in the JSON output. This keeps the pipeline artifacts in sync — `ridl.json` always tracks the ridl.md version it was generated from.

**Local modification suffix:** If `ridl.json` is modified via this skill (e.g., reordering iteration definitions, tweaking criteria) but the source `ridl.md` was **not** regenerated, append a semver pre-release suffix: `-ridljson.N`, where N increments with each local edit. For example:

- First local edit: `1.0.0-ridljson.1`
- Second local edit: `1.0.0-ridljson.2`
- Source ridl.md regenerated: version resets to the new ridl.md version (e.g., `1.0.0-ridlmd.1` or `1.1.0`) with no `-ridljson` suffix

---

## Output Format

```json
{
  "version": "[inherited from source ridl.md]",
  "generatedBy": "ridl-skills:ridljson v[skill version from frontmatter]",
  "project": "[Project Name]",
  "branchName": "ridl/[feature-name-kebab-case]",
  "description": "[Feature description from PRD title/intro]",
  "universalContext": {
    "nonFunctionalRequirements": [
      "NF requirement from source ridl.md"
    ],
    "developerExperience": [
      "DX requirement from source ridl.md"
    ],
    "technicalArchitecture": [
      "Architecture constraint from source ridl.md"
    ]
  },
  "milestones": [
    {
      "id": "MS-1",
      "version": "v0.1",
      "theme": "[Theme]",
      "definitionIds": ["ID-001", "ID-002"]
    }
  ],
  "iterationDefinitions": [
    {
      "id": "ID-001",
      "userStoryTitle": "[Title]",
      "userStoryDescription": "As a [user], I want [feature] so that [benefit]",
      "prdReferences": [
        "TE-1: Summary of referenced requirement",
        "TE-2: Summary of another referenced requirement"
      ],
      "acceptanceCriteria": [
        "Criterion 1",
        "Criterion 2",
        "Typecheck passes"
      ],
      "milestone": "MS-1",
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

---

## Iteration Definition Size: The Number One Rule

**Each iteration definition must be completable in ONE RIDL iteration (one context window).**

RIDL spawns a fresh instance per iteration with no memory of previous work. If an iteration definition is too big, the LLM runs out of context before finishing and produces broken code.

### Right-sized:
- Add a database column and migration
- Add a UI component to an existing page
- Update a server action with new logic
- Add a filter dropdown to a list

### Too big (split these):
- "Build the entire dashboard" — Split into: schema, queries, UI components, filters
- "Add authentication" — Split into: schema, middleware, login UI, session handling
- "Refactor the API" — Split into one iteration definition per endpoint or pattern

**Rule of thumb:** If the change cannot be described in 2-3 sentences, it is too big.

---

## Ordering: Dependencies First

Iteration definitions execute in priority order. Earlier ones must not depend on later ones.

**Correct order:**
1. Schema/database changes (migrations)
2. Server actions / backend logic
3. UI components that use the backend
4. Dashboard/summary views that aggregate data

**Wrong order:**
1. UI component (depends on schema that does not exist yet)
2. Schema change

---

## Acceptance Criteria: Must Be Verifiable

Each criterion must be something the agent can CHECK, not something vague.

### Good criteria (verifiable):
- "Add `status` column to tasks table with default 'pending'"
- "Filter dropdown has options: All, Active, Completed"
- "Clicking delete shows confirmation dialog"
- "Typecheck passes"
- "Tests pass"

### Bad criteria (vague):
- "Works correctly"
- "User can do X easily"
- "Good UX"
- "Handles edge cases"

### Always include as final criterion:
```
"Typecheck passes"
```

For iteration definitions with testable logic, also include:
```
"Tests pass"
```

### For iteration definitions that change UI, also include:
```
"Verify in browser using dev-browser skill"
```

Frontend iteration definitions are NOT complete until visually verified. The agent will use the dev-browser skill to navigate to the page, interact with the UI, and confirm changes work.

---

## Conversion Rules

1. **Each iteration definition becomes one JSON entry** in the `"iterationDefinitions"` array
2. **IDs**: Sequential (ID-001, ID-002, etc.)
3. **Priority**: Based on dependency order, then document order
4. **All iteration definitions**: `passes: false` and empty `notes`
5. **branchName**: Derive from feature name, kebab-case, prefixed with `ridl/`
6. **Always add**: "Typecheck passes" to every iteration definition's acceptance criteria
7. **Universal context**: Convert the `## Universal Context` section from ridl.md into the top-level `"universalContext"` object with `nonFunctionalRequirements`, `developerExperience`, and `technicalArchitecture` arrays
8. **PRD references**: Convert each iteration definition's `**PRD References:**` list into a `"prdReferences"` array of strings (e.g., `"TE-1: User can create a new template"`)
9. **Milestones array**: If the source `ridl.md` has a `## Milestones` section, convert each milestone heading into an entry in the top-level `"milestones"` array with `id`, `version`, `theme`, and `definitionIds`
10. **Per-definition milestone**: Add a `"milestone"` field to each iteration definition object referencing its milestone ID (e.g., `"MS-1"`)
11. **Bidirectional consistency**: Every iteration definition's `"milestone"` value must appear in `milestones[].id`, and every ID in `milestones[].definitionIds` must match an iteration definition with that `"milestone"` value

---

## Splitting Large Features

If a feature is too big for a single iteration definition, split it:

**Original:**
> "Add user notification system"

**Split into:**
1. ID-001: Add notifications table to database
2. ID-002: Create notification service for sending notifications
3. ID-003: Add notification bell icon to header
4. ID-004: Create notification dropdown panel
5. ID-005: Add mark-as-read functionality
6. ID-006: Add notification preferences page

Each is one focused change that can be completed and verified independently.

---

## Example

**Input (`ridl.md`):**
```markdown
# PRD: Task Status Feature

## Introduction

Add ability to mark tasks with different statuses.

## Universal Context

Cross-cutting requirements that apply to ALL iteration definitions.

### Non-Functional Requirements
- App must remain responsive (< 50ms for status updates)

### Developer Experience
- All code must pass linting with zero warnings

### Technical Architecture
- Status values stored as enum type in database

## Iteration Definitions

### ID-001: Add status field to tasks table
**User Story:** As a developer, I need to store task status in the database.

**PRD References:**
- FR-1: Add status tracking to task model

**Acceptance Criteria:**
- [ ] Add status column: 'pending' | 'in_progress' | 'done' (default 'pending')
- [ ] Generate and run migration successfully
- [ ] Typecheck passes

### ID-002: Display status badge on task cards
**User Story:** As a user, I want to see task status at a glance.

**PRD References:**
- FR-2: Display task status visually on cards
- FR-3: Use color-coded status indicators

**Acceptance Criteria:**
- [ ] Each task card shows colored status badge
- [ ] Badge colors: gray=pending, blue=in_progress, green=done
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### ID-003: Add status toggle to task list rows
**User Story:** As a user, I want to change task status directly from the list.

**PRD References:**
- FR-4: Inline status editing from list view

**Acceptance Criteria:**
- [ ] Each row has status dropdown or toggle
- [ ] Changing status saves immediately
- [ ] UI updates without page refresh
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### ID-004: Filter tasks by status
**User Story:** As a user, I want to filter the list to see only certain statuses.

**PRD References:**
- FR-5: Status-based filtering on task list

**Acceptance Criteria:**
- [ ] Filter dropdown: All | Pending | In Progress | Done
- [ ] Filter persists in URL params
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

## Milestones

### MS-1: v0.1 — Status Data Model
- ID-001: Add status field to tasks table
- ID-002: Display status badge on task cards

### MS-2: v0.2 — Status Interactions
- ID-003: Add status toggle to task list rows
- ID-004: Filter tasks by status
```

**Output (`ridl.json`):**
```json
{
  "version": "1.0",
  "generatedBy": "ridl-skills:ridljson v2.0.0",
  "project": "TaskApp",
  "branchName": "ridl/task-status",
  "description": "Task Status Feature - Track task progress with status indicators",
  "universalContext": {
    "nonFunctionalRequirements": [
      "App must remain responsive (< 50ms for status updates)"
    ],
    "developerExperience": [
      "All code must pass linting with zero warnings"
    ],
    "technicalArchitecture": [
      "Status values stored as enum type in database"
    ]
  },
  "milestones": [
    {
      "id": "MS-1",
      "version": "v0.1",
      "theme": "Status Data Model",
      "definitionIds": ["ID-001", "ID-002"]
    },
    {
      "id": "MS-2",
      "version": "v0.2",
      "theme": "Status Interactions",
      "definitionIds": ["ID-003", "ID-004"]
    }
  ],
  "iterationDefinitions": [
    {
      "id": "ID-001",
      "userStoryTitle": "Add status field to tasks table",
      "userStoryDescription": "As a developer, I need to store task status in the database.",
      "prdReferences": [
        "FR-1: Add status tracking to task model"
      ],
      "acceptanceCriteria": [
        "Add status column: 'pending' | 'in_progress' | 'done' (default 'pending')",
        "Generate and run migration successfully",
        "Typecheck passes"
      ],
      "milestone": "MS-1",
      "priority": 1,
      "passes": false,
      "notes": ""
    },
    {
      "id": "ID-002",
      "userStoryTitle": "Display status badge on task cards",
      "userStoryDescription": "As a user, I want to see task status at a glance.",
      "prdReferences": [
        "FR-2: Display task status visually on cards",
        "FR-3: Use color-coded status indicators"
      ],
      "acceptanceCriteria": [
        "Each task card shows colored status badge",
        "Badge colors: gray=pending, blue=in_progress, green=done",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "milestone": "MS-1",
      "priority": 2,
      "passes": false,
      "notes": ""
    },
    {
      "id": "ID-003",
      "userStoryTitle": "Add status toggle to task list rows",
      "userStoryDescription": "As a user, I want to change task status directly from the list.",
      "prdReferences": [
        "FR-4: Inline status editing from list view"
      ],
      "acceptanceCriteria": [
        "Each row has status dropdown or toggle",
        "Changing status saves immediately",
        "UI updates without page refresh",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "milestone": "MS-2",
      "priority": 3,
      "passes": false,
      "notes": ""
    },
    {
      "id": "ID-004",
      "userStoryTitle": "Filter tasks by status",
      "userStoryDescription": "As a user, I want to filter the list to see only certain statuses.",
      "prdReferences": [
        "FR-5: Status-based filtering on task list"
      ],
      "acceptanceCriteria": [
        "Filter dropdown: All | Pending | In Progress | Done",
        "Filter persists in URL params",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "milestone": "MS-2",
      "priority": 4,
      "passes": false,
      "notes": ""
    }
  ]
}
```

---

## Pipeline Context

This skill is step 3 in the RIDL pipeline. All files live in the `ridl/` directory:

```
/ridl-skills:prd      →  ridl/prd.md      (comprehensive PRD)
/ridl-skills:ridlmd   →  ridl/ridl.md     (agent-sized iteration definitions)
/ridl-skills:ridljson  →  ridl/ridl.json   (JSON for autonomous loop)  ← you are here
```

---

## Archiving Previous Runs

**Before writing a new ridl.json, check if there is an existing one from a different feature:**

1. Read the current `ridl/ridl.json` if it exists
2. Check if `branchName` differs from the new feature's branch name
3. If different AND `progress.txt` has content beyond the header:
   - Create archive folder: `ridl/archive/YYYY-MM-DD-feature-name/`
   - Copy current `ridl.json` and `progress.txt` to archive
   - Reset `progress.txt` with fresh header

---

## Checklist Before Saving

Before writing ridl.json, verify:

- [ ] **Previous run archived** (if ridl.json exists with different branchName, archive it first)
- [ ] Universal context extracted into `"universalContext"` object
- [ ] Each iteration definition is completable in one iteration (small enough)
- [ ] Iteration definitions are ordered by dependency (schema to backend to UI)
- [ ] Every iteration definition has "Typecheck passes" as criterion
- [ ] UI iteration definitions have "Verify in browser using dev-browser skill" as criterion
- [ ] Acceptance criteria are verifiable (not vague)
- [ ] No iteration definition depends on a later one
- [ ] Each iteration definition has `"prdReferences"` array
- [ ] Every iteration definition has a `"milestone"` field (if source ridl.md has milestones)
- [ ] Top-level `"milestones"` array is present (if source ridl.md has milestones)
- [ ] Bidirectional consistency: each iteration definition's `"milestone"` matches `milestones[].id`, and each `milestones[].definitionIds` entry matches an iteration definition with that milestone
