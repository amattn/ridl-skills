# CLAUDE.md

## STOP — Required Plugins (check FIRST, before any other work)

The official **skill-creator** plugin MUST be installed. This is a hard requirement — do not proceed with ANY task (reading, editing, reviewing, or discussing skill files) until this is confirmed.

**On every session start, BEFORE doing anything else**, verify that `skill-creator:skill-creator` is available. If it is not:

1. Attempt to install it automatically via `/install skill-creator`
2. If automatic installation fails, STOP and prompt the user: "The skill-creator plugin is required for this project. Please install it by running `/install skill-creator` or adding it manually."
3. Do NOT continue until the plugin is confirmed available.

---

## Project Overview

ridl-skills is a Claude Code plugin providing a 4-step pipeline for generating and converting PRDs into agent-sized iteration definitions and prompt templates for the RIDL (Ralph Iteration Definition List) autonomous agent system.

## Repository Structure

```
skills/
  prd/SKILL.md          - PRD generation skill
  ridlmd/SKILL.md       - PRD → ridl.md conversion skill
  ridljson/SKILL.md     - ridl.md → ridl.json conversion skill
  ridlprompts/SKILL.md  - ridl.json → prompt templates skill
.claude-plugin/
  plugin.json            - Plugin metadata and version
  marketplace.json       - Marketplace listing metadata
```

## Pipeline

```
/ridl-skills:prd        →  ridl/prd.md
/ridl-skills:ridlmd     →  ridl/ridl.md
/ridl-skills:ridljson   →  ridl/ridl.json
/ridl-skills:ridlprompts →  ridl/prompts/*.liquid
```

## Runtime Artifacts

During RIDL execution, the following files are created in `ridl/` alongside the pipeline outputs:
- `progress.md` — Append-only log of what was implemented and verified each iteration
- `learnings.md` — Agent-facing: codebase patterns, tool quirks, implementation techniques, debugging tips. Helps future agents do the work better.
- `emergent.md` — Human-facing: design decisions made without human input, requirement gaps, assumptions needing validation, scope questions, edge cases the PRD didn't anticipate. Items here should be backported to the PRD on regeneration.

All three can be read from and appended to by any agent in any phase (implementation or verification).

## Key Conventions

- **Versioning:** All four skill files and both plugin/marketplace JSON files must stay in sync on version number. Bump all six when releasing. Update `CHANGELOG.md` with the new version entry.
- **Semver rules for skill files:** Patch for small fixes, minor for workflow/template changes, major for breaking output format changes.
- **Frozen iteration definitions:** When all acceptance criteria in an iteration definition have `"status": "pass"` in ridl.json, that iteration definition is frozen and must not be modified. New work is appended as new iteration definitions.
- **Requirement IDs:** Format is `XX-N` (short prefix, hyphen, integer). No letter suffixes.
- **Liquid code blocks:** Use `~~~liquid` fences (not triple backticks) for Liquid template blocks to avoid conflicts with nested markdown code blocks.
- **Commit messages:** Format is `prefix: summary` or `prefix: [vX.Y.Z] summary` when there is a version bump. Follow with a blank line and detailed bullet points describing what changed and why. Common prefixes: `feature`, `fix`, `refactor`, `docs`, `chore`.

## Clarifying Questions Style

When any skill needs to ask the user clarifying questions, use lettered options so users can respond quickly (e.g., "1A, 2C, 3B"). Always ask if the user wants to go through them one by one instead.

```
1. What is the primary platform?
   A. macOS native (Swift/SwiftUI)
   B. iOS (Swift/SwiftUI)
   C. Web (React/Next.js)
   D. Other: [please specify]

2. Who is the primary target user?
   A. End consumers
   B. Internal team / enterprise
   C. Developers / technical users
   D. All of the above
```

Remember to indent the options.

## Development Rules

- Do not modify skill output format without a major version bump.
- Always read a skill file before editing it.
- Test changes by reviewing the skill instructions for internal consistency (e.g., examples match the described format, checklists cover all rules).
