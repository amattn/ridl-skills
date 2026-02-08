---
name: ridljson
description: "Convert ridl.md to ridl.json format for the RIDL autonomous agent system. This skill should be used when the user asks to 'convert ridl.md', 'turn this into ridl json', 'create ridl.json', 'ridl json', 'convert ridl.md to json', or 'make ridl stories from ridl.md'."
user-invocable: true
---

# RIDL PRD Converter

Convert existing PRDs to the ridl.json format that RIDL uses for autonomous execution.

---

## The Job

Read `ridl.md` from `ridl/` (or a path specified by the user) and convert it to `ridl.json` in the same directory.

If no `ridl/ridl.md` is found, prompt the user to either provide a path or run `/ridl-skills:ridlmd` first to generate one.

---

## Output Format

```json
{
  "project": "[Project Name]",
  "branchName": "ridl/[feature-name-kebab-case]",
  "description": "[Feature description from PRD title/intro]",
  "userStories": [
    {
      "id": "US-001",
      "title": "[Story title]",
      "description": "As a [user], I want [feature] so that [benefit]",
      "acceptanceCriteria": [
        "Criterion 1",
        "Criterion 2",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    }
  ]
}
```

---

## Story Size: The Number One Rule

**Each story must be completable in ONE RIDL iteration (one context window).**

RIDL spawns a fresh instance per iteration with no memory of previous work. If a story is too big, the LLM runs out of context before finishing and produces broken code.

### Right-sized stories:
- Add a database column and migration
- Add a UI component to an existing page
- Update a server action with new logic
- Add a filter dropdown to a list

### Too big (split these):
- "Build the entire dashboard" — Split into: schema, queries, UI components, filters
- "Add authentication" — Split into: schema, middleware, login UI, session handling
- "Refactor the API" — Split into one story per endpoint or pattern

**Rule of thumb:** If the change cannot be described in 2-3 sentences, it is too big.

---

## Story Ordering: Dependencies First

Stories execute in priority order. Earlier stories must not depend on later ones.

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

For stories with testable logic, also include:
```
"Tests pass"
```

### For stories that change UI, also include:
```
"Verify in browser using dev-browser skill"
```

Frontend stories are NOT complete until visually verified. The agent will use the dev-browser skill to navigate to the page, interact with the UI, and confirm changes work.

---

## Conversion Rules

1. **Each user story becomes one JSON entry**
2. **IDs**: Sequential (US-001, US-002, etc.)
3. **Priority**: Based on dependency order, then document order
4. **All stories**: `passes: false` and empty `notes`
5. **branchName**: Derive from feature name, kebab-case, prefixed with `ridl/`
6. **Always add**: "Typecheck passes" to every story's acceptance criteria

---

## Splitting Large PRDs

If a PRD has big features, split them:

**Original:**
> "Add user notification system"

**Split into:**
1. US-001: Add notifications table to database
2. US-002: Create notification service for sending notifications
3. US-003: Add notification bell icon to header
4. US-004: Create notification dropdown panel
5. US-005: Add mark-as-read functionality
6. US-006: Add notification preferences page

Each is one focused change that can be completed and verified independently.

---

## Example

**Input (`ridl.md`):**
```markdown
# PRD: Task Status Feature

## Introduction

Add ability to mark tasks with different statuses.

## User Stories

### US-001: Add status field to tasks table
**Description:** As a developer, I need to store task status in the database.

**Acceptance Criteria:**
- [ ] Add status column: 'pending' | 'in_progress' | 'done' (default 'pending')
- [ ] Generate and run migration successfully
- [ ] Typecheck passes

### US-002: Display status badge on task cards
**Description:** As a user, I want to see task status at a glance.

**Acceptance Criteria:**
- [ ] Each task card shows colored status badge
- [ ] Badge colors: gray=pending, blue=in_progress, green=done
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-003: Add status toggle to task list rows
**Description:** As a user, I want to change task status directly from the list.

**Acceptance Criteria:**
- [ ] Each row has status dropdown or toggle
- [ ] Changing status saves immediately
- [ ] UI updates without page refresh
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill

### US-004: Filter tasks by status
**Description:** As a user, I want to filter the list to see only certain statuses.

**Acceptance Criteria:**
- [ ] Filter dropdown: All | Pending | In Progress | Done
- [ ] Filter persists in URL params
- [ ] Typecheck passes
- [ ] Verify in browser using dev-browser skill
```

**Output (`ridl.json`):**
```json
{
  "project": "TaskApp",
  "branchName": "ridl/task-status",
  "description": "Task Status Feature - Track task progress with status indicators",
  "userStories": [
    {
      "id": "US-001",
      "title": "Add status field to tasks table",
      "description": "As a developer, I need to store task status in the database.",
      "acceptanceCriteria": [
        "Add status column: 'pending' | 'in_progress' | 'done' (default 'pending')",
        "Generate and run migration successfully",
        "Typecheck passes"
      ],
      "priority": 1,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-002",
      "title": "Display status badge on task cards",
      "description": "As a user, I want to see task status at a glance.",
      "acceptanceCriteria": [
        "Each task card shows colored status badge",
        "Badge colors: gray=pending, blue=in_progress, green=done",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "priority": 2,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-003",
      "title": "Add status toggle to task list rows",
      "description": "As a user, I want to change task status directly from the list.",
      "acceptanceCriteria": [
        "Each row has status dropdown or toggle",
        "Changing status saves immediately",
        "UI updates without page refresh",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
      "priority": 3,
      "passes": false,
      "notes": ""
    },
    {
      "id": "US-004",
      "title": "Filter tasks by status",
      "description": "As a user, I want to filter the list to see only certain statuses.",
      "acceptanceCriteria": [
        "Filter dropdown: All | Pending | In Progress | Done",
        "Filter persists in URL params",
        "Typecheck passes",
        "Verify in browser using dev-browser skill"
      ],
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
/ridl-skills:ridlmd   →  ridl/ridl.md     (agent-sized user stories)
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
- [ ] Each story is completable in one iteration (small enough)
- [ ] Stories are ordered by dependency (schema to backend to UI)
- [ ] Every story has "Typecheck passes" as criterion
- [ ] UI stories have "Verify in browser using dev-browser skill" as criterion
- [ ] Acceptance criteria are verifiable (not vague)
- [ ] No story depends on a later story
