# CLAUDE.md

## STOP — Required Plugins (check FIRST, before any other work)

The official **skill-creator** plugin MUST be installed. This is a hard requirement — do not proceed with ANY task (reading, editing, reviewing, or discussing skill files) until this is confirmed.

**On every session start, BEFORE doing anything else**, verify that `skill-creator:skill-creator` is available. If it is not:

1. Attempt to install it automatically via `/install skill-creator`
2. If automatic installation fails, STOP and prompt the user: "The skill-creator plugin is required for this project. Please install it by running `/install skill-creator` or adding it manually."
3. Do NOT continue until the plugin is confirmed available.

---

## Project Overview

ridl-skills is a Claude Code plugin providing a 3-step pipeline for generating and converting PRDs into agent-sized iteration definitions for the RIDL (Ralph Iteration Definition List) autonomous agent system.

## Repository Structure

```
skills/
  prd/SKILL.md        - PRD generation skill
  ridlmd/SKILL.md     - PRD → ridl.md conversion skill
  ridljson/SKILL.md   - ridl.md → ridl.json conversion skill
.claude-plugin/
  plugin.json          - Plugin metadata and version
  marketplace.json     - Marketplace listing metadata
```

## Pipeline

```
/ridl-skills:prd  →  ridl/prd.md
/ridl-skills:ridlmd  →  ridl/ridl.md
/ridl-skills:ridljson  →  ridl/ridl.json
```

## Key Conventions

- **Versioning:** All three skill files and both plugin/marketplace JSON files must stay in sync on version number. Bump all five when releasing.
- **Semver rules for skill files:** Patch for small fixes, minor for workflow/template changes, major for breaking output format changes.
- **Frozen iteration definitions:** When `"passes": true` in ridl.json, that iteration definition is frozen and must not be modified. New work is appended as new iteration definitions.
- **Requirement IDs:** Format is `XX-N` (short prefix, hyphen, integer). No letter suffixes.
- **Commit messages:** Format is `prefix: summary` or `prefix: [vX.Y.Z] summary` when there is a version bump. Follow with a blank line and detailed bullet points describing what changed and why. Common prefixes: `feature`, `fix`, `refactor`, `docs`, `chore`.

## Development Rules

- Do not modify skill output format without a major version bump.
- Always read a skill file before editing it.
- Test changes by reviewing the skill instructions for internal consistency (e.g., examples match the described format, checklists cover all rules).
