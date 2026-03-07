# Changelog

All notable changes to ridl-skills are documented in this file.

## [3.0.0] — 2026-03-06

Two-phase iteration workflow and per-criterion status tracking. This is a major structural change to how RIDL iterations execute and how acceptance criteria are tracked in ridl.json.

### Breaking Changes — JSON Format

- **Per-criterion status tracking** — Acceptance criteria are now structured objects `{ "criterion": "...", "status": "..." }` instead of plain strings. Status enum: `not_started`, `fail`, `pass`, `error` — enabling real-time red/green progress tracking.
- **JSON key rename** — `userStoryTitle` → `title`, `userStoryDescription` → `description` in ridl.json iteration definitions.
- **Remove `passes` boolean** — Frozen definition logic is now derived from all criteria having `"status": "pass"`, not a single `passes` flag.

### Breaking Changes — Two-Phase Iteration Workflow

Each iteration definition now goes through two phases across at least two separate agent invocations:

1. **Implementation** (invocation 1) — Agent writes failing tests first, then implements until green, updating per-criterion statuses and runtime files in real time.
2. **Verification** (invocation 2+, fresh context) — Independent agent reviews work, confirms criteria pass, fixes obvious issues in place, updates runtime files with final results, or signals back to implementation for complex problems.

Three verification outcomes:
- **Complete** — All criteria confirmed `"pass"`. Runtime files updated, agent signals `<ridler-verification-complete/>`.
- **Fix in place** — Issues where the fix is clear and obvious. Verification agent adds new acceptance criteria to `ridl/ridl.md` and `ridl/ridl.json`, then fixes with red/green discipline.
- **Back to implementation** — Issues the verification agent cannot fix. Adds new criteria set to `"fail"`, signals `<ridler-verification-failed/>`, and the harness loops back to a fresh implementation invocation.

### Added

- **Harness signals** — `<ridler-implementation-complete/>`, `<ridler-verification-complete/>`, `<ridler-verification-failed/>`, `<ridler-blocked/>` for agent-to-harness communication. Every agent invocation must end with exactly one signal.
- **Emergent items system** — `ridl/emergent.md` with EM-N identifiers, Outcome tracking (`pending` → `approved` | `approved with changes` | `rejected` | `deferred`), and PRD triage workflow for backporting agent discoveries into the PRD.
- **Verification template** — `verification.liquid` for the verification phase.
- **Emergent format template** — `emergent_format.liquid` with Outcome field and category tags.
- **Red/green testing discipline** — Agents must write failing tests before implementation; per-criterion statuses track the red → fail → pass progression in real time.
- **New criteria addition during verification** — Verification agent adds new acceptance criteria to both `ridl/ridl.md` and `ridl/ridl.json` before fixing issues or handing back to implementation, ensuring every fix is tracked.
- **Implementation → Verification retry loop** — When verification fails, the harness loops back to a fresh implementation invocation targeting only the failing criteria, creating a natural retry cycle.
- **CHANGELOG.md** — This file.
- **README overhaul** — Key Concepts, Per-Criterion Status Tracking, Two-Phase Workflow, Implementing a RIDL Harness, Harness Signals, and Emergent Items and Triage sections.
- **CLAUDE.md** — Runtime artifacts section (`progress.md`, `learnings.md`, `emergent.md`), updated frozen definition logic to use per-criterion status.

### Changed

- Template count increased from 6 to 8 (added `verification.liquid` and `emergent_format.liquid`).
- `learnings_report.liquid` renamed to `learnings_format.liquid`.
- `progress_report.liquid` renamed to `progress_format.liquid`.
- `story_context.liquid` renamed to `iteration_context.liquid`.
- Plugin description updated: "PRD generation skills" → "PRD & prompt generation skills".
- PRD triage: deferred items go to Open Questions (not just stay in emergent.md); rejected items go to Resolved Questions with EM-N reference.
- PRD triage ordering: save `ridl/prd.md` first, then update Outcome in `ridl/emergent.md`.
- Frozen definition logic updated across all four skill files to use per-criterion status instead of `passes` boolean.

## [2.2.0] — 2026-03-06

### Added

- **ridlprompts skill** — Fourth pipeline step generating Liquid prompt templates into `ridl/prompts/`.
- **Templates:** `agent_instructions.liquid`, `story_context.liquid`, `progress_report.liquid`, `learnings_report.liquid`, `edit_file.liquid`, `create_file.liquid`.
- Smart merge behavior for existing templates (overwrite unmodified, merge customized with user review).
- `ridl/learnings.md` — Separate file for agent-facing knowledge (patterns, gotchas, techniques), split from progress.md.
- `ridl/progress.md` format updated to track: what was implemented, files changed, verification proof.
- Generated-by lines with skill version in templates.

### Changed

- Pipeline expanded from 3 steps to 4 (added ridlprompts).
- CLAUDE.md updated for 4-step pipeline and `~~~liquid` fence convention.

## [2.1.1] — 2026-03-05

### Added

- **Frozen iteration definitions** — When `passes: true`, the iteration definition is locked and must not be modified. New work is appended as new iteration definitions.

## [2.1.0] — 2026-03-03

### Added

- Milestone `name`, `theme` (optional), and `version` fields.
- Enforced `XX-N` requirement ID format (no letter suffixes).

## [2.0.0] — 2026-02-11

### Breaking Changes

- **Iteration definitions** replace user stories as the unit of work. Each encompasses a user story, PRD references, and acceptance criteria.
- JSON keys: `iterationDefinitions`, `userStoryTitle`, `userStoryDescription`.
- `universalContext` and `prdReferences` added to JSON schema.

### Added

- **Universal Context** — Cross-cutting constraints (NF, DX, architecture) extracted from the PRD and applied to all iteration definitions.
- **PRD References** — Each iteration definition traces back to specific requirement codes.
- `ID-###` prefix for iteration definitions.
- `Generated by` field in all pipeline outputs.

### Changed

- ridlmd and ridljson restructured around iteration definitions.
- prd skill bumped to v1.1.0 (added Generated by header).

## [1.1.0] — 2026-02-09

### Added

- **Milestone awareness** — ridlmd generates a Milestones section; ridljson adds top-level `milestones` array and per-definition `milestone` field with bidirectional consistency checks.
- **Semantic versioning** — All skills carry a `version` field in YAML frontmatter. Generated artifacts inherit upstream version with `-ridlmd.N` / `-ridljson.N` suffix for local edits.

## [1.0.0] — 2026-02-09

### Added

- **Testing & verification strategy** — PRD section 12 with red/green discipline, coverage expectations, and verification commands.
- **Build identification & version tracking** — DX section 6.3 requiring version display (CLI, GUI, web, logs) and semver-during-development rules.
- **CLAUDE.md** — Project conventions, required plugins, development rules.

## [0.1.0] — 2026-02-07

Initial release.

- **prd skill** — Generate comprehensive PRDs with requirements tables, technical architecture, user flows, and release milestones.
- **ridlmd skill** — Convert PRDs to agent-sized iteration definitions.
- **ridljson skill** — Convert ridl.md to ridl.json for autonomous execution.
