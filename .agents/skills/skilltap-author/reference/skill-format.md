# Skill Format Reference

A skill is a directory containing a `SKILL.md` file. The frontmatter tells skilltap and the agent what the skill does and when to load it. Everything below the frontmatter is the body — the actual instructions the agent reads.

## Frontmatter schema

Source of truth: `packages/core/src/schemas/skill.ts` (`SkillFrontmatterSchema`).

```yaml
---
name: <kebab-case-name>     # required. 1–64 chars. /^[a-z0-9]+(-[a-z0-9]+)*$/
description: <paragraph>    # required. 1–1024 chars. Action-oriented. See guidance below.
license: MIT                # optional. Any SPDX identifier.
compatibility: <notes>      # optional. Max 500 chars. Agent compatibility notes (e.g. "Claude Code only").
metadata:                   # optional. Arbitrary key-value pairs.
  key: value
---
```

`skilltap doctor skill <path>` enforces all constraints. The `name` field must also match the parent directory name exactly — validation fails if they differ.

The frontmatter parser supports YAML-style `---`...`---` delimiters with simple scalars, block scalars (`>` folded, `|` literal), and one level of nested objects. Do not use YAML anchors, aliases, or complex types.

## Crafting the `description` field

The description is the primary trigger for agent skill-loading. Make it exhaustive about what actions and topics it covers.

**Good** — action-oriented, lists synonyms and trigger phrases:
```
Manage agent skills, plugins, and standalone MCP servers via the skilltap
CLI — install, update, remove, list, toggle, adopt, move, configure, and
inspect. Use when the user wants to install / update / remove / configure
skilltap-managed content, work with skilltap.toml, or troubleshoot
state.json.
```

**Bad** — too vague to trigger reliably:
```
Skill management.
```

Rules:
- Lead with verbs: "Create, scaffold, install, validate..."
- List alternate phrasings a user might say.
- Include the tool name if it's not obvious from context.
- For narrow skills, include representative trigger phrases in plain language.

## File layout convention

```
<skill-name>/
├── SKILL.md                  # required — at the directory root
├── reference/                # optional — additional docs loaded on demand
│   └── *.md                  # no frontmatter needed on reference pages
├── scripts/                  # optional — executable helpers
│   └── *.sh / *.js / *.py
└── README.md                 # optional — for humans browsing the repo
```

Reference pages (`reference/*.md`) do not need frontmatter. The agent loads them explicitly when needed rather than at skill-load time — ideal for dense schemas, examples, and lookup tables that would bloat the main file.

## Multi-skill repos

A single repo can contain many skills. Discovery (`packages/core/src/scanner.ts`) finds skills via this priority order:

1. **Root**: `SKILL.md` at repo root → standalone single-skill repo.
2. **Standard**: `.agents/skills/*/SKILL.md` → each match is a skill, named by parent directory.
3. **Skills directory**: `skills/SKILL.md` (flat) or `skills/*/SKILL.md`.
4. **Plugin directory**: `plugins/*/skills/*/SKILL.md` (Claude Code plugin convention).
5. **Agent-specific**: `.claude/skills/*/SKILL.md`, `.cursor/skills/*/SKILL.md`, `.codex/skills/*/SKILL.md`, `.gemini/skills/*/SKILL.md`, `.windsurf/skills/*/SKILL.md`.
6. **Deep scan**: `**/SKILL.md` anywhere else. Triggers a confirmation prompt unless `--yes`.

Steps 1–5 are checked first. If any of them find skills, step 6 is skipped. If the same SKILL.md is found via multiple paths, deduplication prefers the `.agents/skills/` path.

In TTY mode, multi-skill installs prompt the user to pick. With `--yes` (or non-TTY mode), all discovered skills are auto-selected.

If you have a multi-skill repo, each skill directory name MUST match its `frontmatter.name`. Use `--template multi` to scaffold the layout.

## Complete example

Copy and adapt as your starting point:

```markdown
---
name: my-tool
description: >
  Use, configure, and troubleshoot my-tool — a CLI for X. Load when the user
  asks about installing my-tool, running my-tool commands, debugging my-tool
  errors, or configuring my-tool settings.
license: MIT
compatibility: Claude Code, Cursor
---

## Overview

Brief description of what the tool does.

## Installation

\`\`\`bash
npm install -g my-tool
\`\`\`

## Common commands

| Command | Description |
|---|---|
| `my-tool init` | Initialize a new project |
| `my-tool run`  | Run the project |

## Configuration

Config lives at `~/.my-tool/config.toml`. Key fields:

- `api_key` — required. Your API token.
- `timeout` — optional. Default: 30s.

## Rules

- Always run `my-tool validate` before `my-tool deploy`.
- Prefer `--dry-run` for destructive operations.
```

## Validation checklist

Run `skilltap doctor skill <path>` to validate. Checks:

- `SKILL.md` exists at the path.
- Frontmatter parses and validates against `SkillFrontmatterSchema`.
- `frontmatter.name` matches the parent directory name.
- Static security scan clean (no anti-trojan-source warnings, no dangerous patterns).
- Skill directory size within the configured limit (default 50 KB; configurable via `[scanner].max_size`).
