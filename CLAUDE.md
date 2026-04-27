# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A personal **Claude Code plugin marketplace** containing reusable skills and subagents for automating documentation and design review workflows. Plugins are distributed via Git, with no build pipeline or external dependencies.

## Repository Structure

```
.claude-plugin/marketplace.json    # Root marketplace catalog (lists all plugins)
<plugin-name>/
  .claude-plugin/plugin.json       # Plugin metadata (name, version, description, author)
  skills/<skill-name>/SKILL.md     # Skill definition (markdown with frontmatter)
  agents/<agent-name>.md           # Subagent definitions (optional)
  templates/                       # Shared templates (optional)
```

## Plugin System Conventions

### Plugin Metadata (`plugin.json`)
```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "...",
  "skills": [{ "name": "...", "path": "skills/..." }],
  "agents": [{ "name": "...", "path": "agents/..." }]
}
```

### Skill Files (`SKILL.md`)
Markdown file with a YAML frontmatter block followed by the skill instructions:
```markdown
---
name: skill-name
description: one-line description shown to users
---
<skill instructions in markdown>
```

### Agent Files (`agents/*.md`)
Same frontmatter format as skills, with an additional `model` field:
```markdown
---
name: agent-name
description: ...
model: claude-haiku-4-5 | claude-sonnet-4-6
---
<agent instructions>
```

Use **Haiku** for high-volume/fast agents (explorers, parsers), **Sonnet** for analytical/writing agents.

### Marketplace Index (`.claude-plugin/marketplace.json`)
When adding a new plugin, register it in the root marketplace:
```json
{
  "plugins": [
    {
      "name": "plugin-name",
      "version": "1.0.0",
      "description": "...",
      "source": "./plugin-name",
      "author": { "name": "tnabet", "email": "tnabet@hellowork.com" }
    }
  ]
}
```

## Existing Plugins

### `generate-architecture` (v1.0.2)
Generates `ARCHITECTURE.md` for any codebase using a MapReduce pattern:
1. Orchestrator (skill) detects languages and major modules
2. `repo-explorer` subagents (Haiku) map each module via LSP without reading all code
3. `architecture-writer` subagent (Sonnet) compiles reports into the template

**Critical constraint:** all file paths in output must be complete and relative from project root — no ellipsis or truncation.

### `reviewing-design-specs` (v1.1.0)
Structured review of technical design specs. Accepts Confluence pages (via `acli`), Markdown files, or pasted text. Outputs findings by severity: S1 Blocking / S2 Important / S3 Suggestion.

## Versioning

Follow semantic versioning. Bump the version in both `plugin.json` and `marketplace.json` when publishing changes. Commit message convention: `feat(<plugin-name>): ...` / `fix(<plugin-name>): ...` / `chore: bump <plugin-name> to vX.Y.Z`.
