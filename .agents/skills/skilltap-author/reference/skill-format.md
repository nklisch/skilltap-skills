# Skill Format Reference

A skill is a directory containing a `SKILL.md` file. The frontmatter tells skilltap and the agent what the skill does and when to load it. Everything below the frontmatter is the body — the actual instructions the agent reads.

## Frontmatter schema

Source of truth: `packages/core/src/schemas/skill.ts`.

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

`skilltap verify` enforces all constraints. The `name` field must also match the directory name exactly — verification fails if they differ.

The frontmatter parser supports YAML-style `---`...`---` delimiters with simple scalars, block scalars (`>` folded, `|` literal), and one level of nested objects. Do not use YAML anchors, aliases, or complex types.

## Crafting the `description` field

The description is the primary trigger for agent skill-loading. Make it exhaustive about what actions and topics it covers.

**Good** — action-oriented, lists synonyms and trigger phrases:
```
Manage agent skills and plugins with the skilltap CLI — install, update,
remove, list, adopt, move, disable, enable, link, unlink agent skills and
plugins. Use when the user wants to install / update / remove / configure /
inspect skilltap-managed content.
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

A single repo can contain many skills. Discovery (`packages/core/src/scanner.ts`) finds skills via:

1. `SKILL.md` at repo root — single-skill repo.
2. `<dir>/SKILL.md` at any depth — each directory becomes a separate installable skill.
3. `plugins/*/skills/*/SKILL.md` — Claude Code marketplace layout.

In agent mode, multi-skill installs auto-select all skills. In interactive mode, the user picks.

If you have a multi-skill repo, each skill directory name must match its `frontmatter.name`. Use `--template multi` to scaffold the layout.

## Complete example

Copy and adapt this as your starting point:

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

```bash
npm install -g my-tool
```

## Common commands

| Command | Description |
|---------|-------------|
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
```
