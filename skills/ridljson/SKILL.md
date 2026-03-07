---
name: ridljson
version: 3.0.0
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
    ],
    "testingAndVerification": [
      "Red/green discipline: new tests must fail before implementation, pass after",
      "Verification command from source ridl.md (e.g., pytest)",
      "Test coverage expectation from source ridl.md"
    ]
  },
  "milestones": [
    {
      "id": "MS-1",
      "name": "[Name]",
      "version": "v0.1",
      "theme": "[Optional longer theme description]",
      "definitionIds": ["ID-001", "ID-002"]
    }
  ],
  "iterationDefinitions": [
    {
      "id": "ID-001",
      "title": "[Title]",
      "description": "As a [user], I want [feature] so that [benefit]",
      "prdReferences": [
        "TE-1: Summary of referenced requirement",
        "TE-2: Summary of another referenced requirement"
      ],
      "acceptanceCriteria": [
        { "criterion": "Criterion 1", "status": "not_started" },
        { "criterion": "Criterion 2", "status": "not_started" },
        { "criterion": "Typecheck passes", "status": "not_started" }
      ],
      "milestone": "MS-1",
      "priority": 1,
      "notes": ""
    }
  ]
}
```

### Acceptance Criteria Status

Each acceptance criterion is an object with a `"criterion"` string and a `"status"` enum. The status tracks real-time progress through the red/green testing workflow:

| Status | Meaning |
|--------|---------|
| `"not_started"` | Criterion has not been attempted yet |
| `"fail"` | Criterion was tested and is currently failing — this is the expected "red" state in red/green testing, distinct from not_started |
| `"pass"` | Criterion has been verified and is passing |
| `"error"` | Unexpected error — criterion could not be evaluated (e.g., build broken, test infrastructure failure) |

The harness updates these statuses in real time as the agent works through each criterion. This gives visibility into iteration progress without waiting for the entire iteration to complete.

---

## Two-Phase Workflow

Each iteration definition goes through two phases across at least two separate agent invocations:

1. **Implementation** — Initial work with red/green tests. The agent writes failing tests first, then implements until green. Updates runtime files (ridl/progress.md, ridl/learnings.md, ridl/emergent.md) as things come up.
2. **Verification** — Independent check in a fresh context window. A second agent invocation reviews the work, runs independent checks, confirms all criteria pass, and updates runtime files with final results. May require multiple invocations if issues are found.

Verification ends in one of three outcomes: **(1) complete** — all criteria pass, runtime files updated, `<ridler-verification-complete/>`; **(2) fix in place** — only when the fix is clear and obvious; verification agent adds new acceptance criteria to `ridl/ridl.md` and `ridl/ridl.json` describing the issue, then fixes with red/green discipline, then completes; **(3) back to implementation** — issues the verification agent cannot fix, adds new acceptance criteria set to `"status": "fail"`, signals `<ridler-verification-failed/>`, and the harness loops back to a fresh implementation invocation targeting the failing criteria. This creates a natural retry loop: Implementation → Verification → (if fail) → Implementation → Verification → ... until all criteria reach `"pass"`.

Agents signal completion state to the harness via `<ridler-implementation-complete/>` (implementation done, move to verification), `<ridler-verification-complete/>` (verification confirmed all pass), `<ridler-verification-failed/>` (needs retry), or `<ridler-blocked/>` (needs human). The workflow details and prompt templates live in `/ridl-skills:ridlprompts`. The JSON format supports this by tracking per-criterion status rather than a single pass/fail flag — the harness can observe the implementation agent setting criteria to `"fail"` then `"pass"`, and the verification agent independently confirming them.

---

## Frozen Iteration Definitions

When converting or updating `ridl.json`, check if a `ridl/ridl.json` already exists. If it does, read it and identify any iteration definitions where **all** acceptance criteria have `"status": "pass"`.

**Frozen iteration definitions must not be modified.** A fully-passing iteration definition represents completed, verified work. Its `title`, `description`, `prdReferences`, `acceptanceCriteria`, `milestone`, and `priority` are all locked.

When updating `ridl/ridl.json`:

1. **Preserve frozen definitions exactly as-is** — copy them verbatim into the new output, including all `"status": "pass"` criteria and any `"notes"` content
2. **Never merge new requirements into frozen definitions** — if a requirement change or new feature would have affected a frozen definition, it must become a new iteration definition that builds on top of the frozen one
3. **ID numbering continues** — new iteration definitions use the next available ID number after all existing definitions
4. **Frozen definitions keep their position** — they remain at their original priority/order; new definitions are appended after all existing ones
5. **Non-frozen definitions may be updated** — iteration definitions where any criterion is not `"pass"` can still be modified, reordered, split, or merged as needed during regeneration

If no existing `ridl/ridl.json` is found, generate all iteration definitions fresh with all criteria set to `"status": "not_started"`.

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

### Always include on every iteration definition:
```
"Tests written for [requirement IDs] (failing before implementation, passing after)"
"Full test suite passes with no regressions"
"[format check command] produces zero changes"
"[lint command] passes with zero warnings"
"[typecheck command] passes"
"[build command] succeeds with no errors or warnings"
```

Use the exact verification commands from the source ridl.md's Testing & Verification universal context section.

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
4. **Key names**: Use `"title"` (not `"userStoryTitle"`) and `"description"` (not `"userStoryDescription"`)
5. **New iteration definitions**: All criteria set to `"status": "not_started"` and empty `"notes"`; **frozen definitions** (all criteria `"status": "pass"` in existing ridl/ridl.json) are preserved exactly as-is
6. **branchName**: Derive from feature name, kebab-case, prefixed with `ridl/`
7. **Always add**: All 5 verification commands (test, format, lint, typecheck, build) from the source ridl.md's universal context to every iteration definition's acceptance criteria
8. **Universal context**: Convert the `## Universal Context` section from ridl.md into the top-level `"universalContext"` object with `nonFunctionalRequirements`, `developerExperience`, `technicalArchitecture`, and `testingAndVerification` arrays
9. **PRD references**: Convert each iteration definition's `**PRD References:**` list into a `"prdReferences"` array of strings (e.g., `"TE-1: User can create a new template"`)
10. **Milestones array**: If the source `ridl.md` has a `## Milestones` section, convert each milestone heading into an entry in the top-level `"milestones"` array with `id`, `name` (required), `version` (required), `theme` (optional — include only if the source milestone has a blockquote theme line), and `definitionIds`
11. **Per-definition milestone**: Add a `"milestone"` field to each iteration definition object referencing its milestone ID (e.g., `"MS-1"`)
12. **Bidirectional consistency**: Every iteration definition's `"milestone"` value must appear in `milestones[].id`, and every ID in `milestones[].definitionIds` must match an iteration definition with that `"milestone"` value

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

### Testing & Verification
- Red/green discipline: new tests must fail before implementation, pass after
- Verification commands: `pytest` (test suite), `ruff format --check .` (formatter), `ruff check .` (linter), `mypy .` (typecheck), `python -m build` (build)
- Test names include requirement IDs (e.g., test_FR_1_status_tracking)
- Unit tests for every new function; integration tests at component boundaries

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
- [ ] Tests written for FR-1 (failing before implementation, passing after)
- [ ] Full test suite passes with no regressions
- [ ] `ruff format --check .` produces zero changes
- [ ] `ruff check .` passes with zero warnings
- [ ] `mypy .` passes
- [ ] `python -m build` succeeds with no errors or warnings

### ID-002: Display status badge on task cards
**User Story:** As a user, I want to see task status at a glance.

**PRD References:**
- FR-2: Display task status visually on cards
- FR-3: Use color-coded status indicators

**Acceptance Criteria:**
- [ ] Each task card shows colored status badge
- [ ] Badge colors: gray=pending, blue=in_progress, green=done
- [ ] Tests written for FR-2 and FR-3 (failing before implementation, passing after)
- [ ] Full test suite passes with no regressions
- [ ] `ruff format --check .` produces zero changes
- [ ] `ruff check .` passes with zero warnings
- [ ] `mypy .` passes
- [ ] `python -m build` succeeds with no errors or warnings
- [ ] Verify in browser using dev-browser skill

### ID-003: Add status toggle to task list rows
**User Story:** As a user, I want to change task status directly from the list.

**PRD References:**
- FR-4: Inline status editing from list view

**Acceptance Criteria:**
- [ ] Each row has status dropdown or toggle
- [ ] Changing status saves immediately
- [ ] UI updates without page refresh
- [ ] Tests written for FR-4 (failing before implementation, passing after)
- [ ] Full test suite passes with no regressions
- [ ] `ruff format --check .` produces zero changes
- [ ] `ruff check .` passes with zero warnings
- [ ] `mypy .` passes
- [ ] `python -m build` succeeds with no errors or warnings
- [ ] Verify in browser using dev-browser skill

### ID-004: Filter tasks by status
**User Story:** As a user, I want to filter the list to see only certain statuses.

**PRD References:**
- FR-5: Status-based filtering on task list

**Acceptance Criteria:**
- [ ] Filter dropdown: All | Pending | In Progress | Done
- [ ] Filter persists in URL params
- [ ] Tests written for FR-5 (failing before implementation, passing after)
- [ ] Full test suite passes with no regressions
- [ ] `ruff format --check .` produces zero changes
- [ ] `ruff check .` passes with zero warnings
- [ ] `mypy .` passes
- [ ] `python -m build` succeeds with no errors or warnings
- [ ] Verify in browser using dev-browser skill

## Milestones

### MS-1: v0.1 — Status Data Model
> Establish the foundational data layer for task status tracking
- ID-001: Add status field to tasks table
- ID-002: Display status badge on task cards

### MS-2: v0.2 — Status Interactions
> Enable users to interact with and filter by task status
- ID-003: Add status toggle to task list rows
- ID-004: Filter tasks by status
```

**Output (`ridl.json`):**
```json
{
  "version": "1.0",
  "generatedBy": "ridl-skills:ridljson v3.0.0",
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
    ],
    "testingAndVerification": [
      "Red/green discipline: new tests must fail before implementation, pass after",
      "Verification commands: pytest (test suite), ruff format --check . (formatter), ruff check . (linter), mypy . (typecheck), python -m build (build)",
      "Test names include requirement IDs (e.g., test_FR_1_status_tracking)",
      "Unit tests for every new function; integration tests at component boundaries"
    ]
  },
  "milestones": [
    {
      "id": "MS-1",
      "name": "Status Data Model",
      "version": "v0.1",
      "theme": "Establish the foundational data layer for task status tracking",
      "definitionIds": ["ID-001", "ID-002"]
    },
    {
      "id": "MS-2",
      "name": "Status Interactions",
      "version": "v0.2",
      "theme": "Enable users to interact with and filter by task status",
      "definitionIds": ["ID-003", "ID-004"]
    }
  ],
  "iterationDefinitions": [
    {
      "id": "ID-001",
      "title": "Add status field to tasks table",
      "description": "As a developer, I need to store task status in the database.",
      "prdReferences": [
        "FR-1: Add status tracking to task model"
      ],
      "acceptanceCriteria": [
        { "criterion": "Add status column: 'pending' | 'in_progress' | 'done' (default 'pending')", "status": "not_started" },
        { "criterion": "Generate and run migration successfully", "status": "not_started" },
        { "criterion": "Tests written for FR-1 (failing before implementation, passing after)", "status": "not_started" },
        { "criterion": "Full test suite passes with no regressions", "status": "not_started" },
        { "criterion": "ruff format --check . produces zero changes", "status": "not_started" },
        { "criterion": "ruff check . passes with zero warnings", "status": "not_started" },
        { "criterion": "mypy . passes", "status": "not_started" },
        { "criterion": "python -m build succeeds with no errors or warnings", "status": "not_started" }
      ],
      "milestone": "MS-1",
      "priority": 1,
      "notes": ""
    },
    {
      "id": "ID-002",
      "title": "Display status badge on task cards",
      "description": "As a user, I want to see task status at a glance.",
      "prdReferences": [
        "FR-2: Display task status visually on cards",
        "FR-3: Use color-coded status indicators"
      ],
      "acceptanceCriteria": [
        { "criterion": "Each task card shows colored status badge", "status": "not_started" },
        { "criterion": "Badge colors: gray=pending, blue=in_progress, green=done", "status": "not_started" },
        { "criterion": "Tests written for FR-2 and FR-3 (failing before implementation, passing after)", "status": "not_started" },
        { "criterion": "Full test suite passes with no regressions", "status": "not_started" },
        { "criterion": "ruff format --check . produces zero changes", "status": "not_started" },
        { "criterion": "ruff check . passes with zero warnings", "status": "not_started" },
        { "criterion": "mypy . passes", "status": "not_started" },
        { "criterion": "python -m build succeeds with no errors or warnings", "status": "not_started" },
        { "criterion": "Verify in browser using dev-browser skill", "status": "not_started" }
      ],
      "milestone": "MS-1",
      "priority": 2,
      "notes": ""
    },
    {
      "id": "ID-003",
      "title": "Add status toggle to task list rows",
      "description": "As a user, I want to change task status directly from the list.",
      "prdReferences": [
        "FR-4: Inline status editing from list view"
      ],
      "acceptanceCriteria": [
        { "criterion": "Each row has status dropdown or toggle", "status": "not_started" },
        { "criterion": "Changing status saves immediately", "status": "not_started" },
        { "criterion": "UI updates without page refresh", "status": "not_started" },
        { "criterion": "Tests written for FR-4 (failing before implementation, passing after)", "status": "not_started" },
        { "criterion": "Full test suite passes with no regressions", "status": "not_started" },
        { "criterion": "ruff format --check . produces zero changes", "status": "not_started" },
        { "criterion": "ruff check . passes with zero warnings", "status": "not_started" },
        { "criterion": "mypy . passes", "status": "not_started" },
        { "criterion": "python -m build succeeds with no errors or warnings", "status": "not_started" },
        { "criterion": "Verify in browser using dev-browser skill", "status": "not_started" }
      ],
      "milestone": "MS-2",
      "priority": 3,
      "notes": ""
    },
    {
      "id": "ID-004",
      "title": "Filter tasks by status",
      "description": "As a user, I want to filter the list to see only certain statuses.",
      "prdReferences": [
        "FR-5: Status-based filtering on task list"
      ],
      "acceptanceCriteria": [
        { "criterion": "Filter dropdown: All | Pending | In Progress | Done", "status": "not_started" },
        { "criterion": "Filter persists in URL params", "status": "not_started" },
        { "criterion": "Tests written for FR-5 (failing before implementation, passing after)", "status": "not_started" },
        { "criterion": "Full test suite passes with no regressions", "status": "not_started" },
        { "criterion": "ruff format --check . produces zero changes", "status": "not_started" },
        { "criterion": "ruff check . passes with zero warnings", "status": "not_started" },
        { "criterion": "mypy . passes", "status": "not_started" },
        { "criterion": "python -m build succeeds with no errors or warnings", "status": "not_started" },
        { "criterion": "Verify in browser using dev-browser skill", "status": "not_started" }
      ],
      "milestone": "MS-2",
      "priority": 4,
      "notes": ""
    }
  ]
}
```

---

## Pipeline Context

This skill is step 3 in the RIDL pipeline. All files live in the `ridl/` directory:

```
/ridl-skills:prd         →  ridl/prd.md             (comprehensive PRD)
/ridl-skills:ridlmd      →  ridl/ridl.md            (agent-sized iteration definitions)
/ridl-skills:ridljson    →  ridl/ridl.json          (JSON for autonomous loop)  ← you are here
/ridl-skills:ridlprompts →  ridl/prompts/*.liquid   (harness prompt templates)
```

---

## Archiving Previous Runs

**Before writing a new ridl/ridl.json, check if there is an existing one from a different feature:**

1. Read the current `ridl/ridl.json` if it exists
2. Check if `branchName` differs from the new feature's branch name
3. If different AND `ridl/progress.md` has content beyond the header:
   - Create archive folder: `ridl/archive/YYYY-MM-DD-feature-name/`
   - Copy current `ridl/ridl.json`, `ridl/progress.md`, `ridl/learnings.md`, and `ridl/emergent.md` to archive
   - Reset `ridl/progress.md`, `ridl/learnings.md`, and `ridl/emergent.md` with fresh headers

---

## Checklist Before Saving

Before writing ridl/ridl.json, verify:

- [ ] **Previous run archived** (if ridl/ridl.json exists with different branchName, archive it first)
- [ ] **Frozen definitions preserved** — all iteration definitions where every criterion has `"status": "pass"` in existing ridl/ridl.json copied verbatim
- [ ] **No frozen definitions modified** — new requirements added as new iteration definitions, not merged into frozen ones
- [ ] Universal context extracted into `"universalContext"` object (including `testingAndVerification` array)
- [ ] Each iteration definition is completable in one iteration (small enough)
- [ ] Iteration definitions are ordered by dependency (schema to backend to UI)
- [ ] Every iteration definition has test-writing criteria (tests for new functionality, red/green, full suite passes)
- [ ] Every iteration definition lists all 5 verification commands from universal context (test, format, lint, typecheck, build)
- [ ] UI iteration definitions have "Verify in browser using dev-browser skill" as criterion
- [ ] Acceptance criteria are verifiable (not vague)
- [ ] Acceptance criteria use object format with `"criterion"` and `"status"` fields
- [ ] All new criteria initialized to `"status": "not_started"`
- [ ] No iteration definition depends on a later one
- [ ] Each iteration definition has `"prdReferences"` array
- [ ] Every iteration definition has a `"milestone"` field (if source ridl.md has milestones)
- [ ] Top-level `"milestones"` array is present (if source ridl.md has milestones)
- [ ] Bidirectional consistency: each iteration definition's `"milestone"` matches `milestones[].id`, and each `milestones[].definitionIds` entry matches an iteration definition with that milestone
- [ ] Keys use `"title"` and `"description"` (not legacy `"userStoryTitle"` / `"userStoryDescription"`)
- [ ] No `"passes"` boolean on iteration definitions (status is tracked per-criterion)
