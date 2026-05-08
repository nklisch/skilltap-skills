# Plugin Format Reference

A plugin bundles skills, MCP servers, and agent definitions into a single installable unit. When a user runs `skilltap plugin install <url>`, skilltap detects the format, parses the manifest, injects MCP servers into their agent config, copies skills to `.agents/skills/`, and places agent definitions where the agent expects them.

## Detection files

| Format | Manifest location |
|--------|------------------|
| Claude Code | `.claude-plugin/plugin.json` |
| Codex | `.codex-plugin/plugin.json` |

skilltap checks for Claude Code first, then Codex. Both normalize to the same internal `PluginManifest` structure.

## Claude Code `plugin.json`

Source: `packages/core/src/schemas/plugin.ts` (`ClaudePluginJsonSchema`).

**Required:** `name`

**Optional:**

| Field | Type | Notes |
|-------|------|-------|
| `description` | string | One-line summary |
| `version` | string | SemVer recommended |
| `author` | `{ name, email?, url? }` | Author info |
| `homepage` | string | URL |
| `repository` | string | URL |
| `license` | string | SPDX identifier |
| `keywords` | string[] | Searchable tags |
| `skills` | string \| string[] \| `{ path, description? }`[] | Skill paths relative to plugin root. Omit → auto-scans `skills/*/SKILL.md` |
| `commands` | string \| string[] | Slash command paths (platform-specific) |
| `agents` | string \| string[] \| `{ path, name? }`[] | Agent `.md` file paths. Omit → auto-scans `agents/*.md` |
| `mcpServers` | string \| string[] \| `{ name: server }` | Path(s) to `.mcp.json` or inline server map. Omit → checks `.mcp.json` at plugin root |

**Ignored** (platform-specific, not portable): `hooks`, `lspServers`, `outputStyles`, `channels`, `userConfig`.

Unknown fields pass through (the schema uses `.passthrough()`).

## Codex `plugin.json`

Source: `packages/core/src/schemas/plugin.ts` (`CodexPluginJsonSchema`).

**Required:** `name`, `version`, `description`

**Optional:** same shape as Claude Code except:
- `skills` — string (single path) only, not array or object form
- `mcpServers` — string (path to `.mcp.json`) only, not inline object
- No `agents`, `commands` fields
- `apps` — present but ignored
- `interface` — present but ignored

Codex plugins do not support agent definitions.

## MCP server schemas

Both formats support `stdio` and `http` server types. The `type` field defaults to `"stdio"` if omitted.

### Stdio server

```json
{
  "type": "stdio",
  "command": "npx",
  "args": ["-y", "@modelcontextprotocol/server-postgres"],
  "env": {
    "DATABASE_URL": "postgres://localhost/mydb"
  }
}
```

Required: `command`. Optional: `args` (default `[]`), `env` (default `{}`).

### HTTP server (added v0.10.8)

```json
{
  "type": "http",
  "url": "https://api.example.com/mcp",
  "headers": {
    "Authorization": "Bearer ${API_TOKEN}"
  }
}
```

Required: `url`. Optional: `headers` (default `{}`).

### Inline vs. file-based MCP

Inline in `plugin.json` (Claude Code only):
```json
{
  "mcpServers": {
    "database": { "command": "npx", "args": ["-y", "@mcp/postgres"] },
    "api":      { "type": "http", "url": "https://api.example.com/mcp" }
  }
}
```

File-based (both formats) — path relative to plugin root:
```json
{
  "mcpServers": ".mcp.json"
}
```

`.mcp.json` supports two on-disk layouts:

**Flat:**
```json
{
  "database": { "command": "npx", "args": ["-y", "@mcp/postgres"] }
}
```

**Wrapped:**
```json
{
  "mcpServers": {
    "database": { "command": "npx", "args": ["-y", "@mcp/postgres"] }
  }
}
```

## Variable substitution

At injection time, these variables are expanded in `command`, `args`, `env` values, `url`, and `headers`:

| Variable | Expands to |
|----------|-----------|
| `${CLAUDE_PLUGIN_ROOT}` | Absolute path to the plugin install root |
| `${CLAUDE_PLUGIN_DATA}` | Plugin data directory |

Use them for paths that must resolve relative to the install location:
```json
{
  "command": "${CLAUDE_PLUGIN_ROOT}/bin/server",
  "args": ["--data", "${CLAUDE_PLUGIN_DATA}"]
}
```

## Namespacing

After injection, MCP server entries appear in the user's per-agent config as:

```
skilltap:<plugin-name>:<server-name>
```

Example: plugin `dev-toolkit`, server `database` → `skilltap:dev-toolkit:database`.

## Agent definition files

Place agent definitions at `agents/<name>.md` in the plugin root (or specify explicit paths in `plugin.json` via the `agents` field).

Format: same SKILL.md frontmatter (`---`...`---`) plus markdown body. The `name` field is optional and defaults to the filename stem.

On install, Claude Code agents are placed at `~/.claude/agents/<name>.md`. Currently only Claude Code consumes agent definitions.

Example `agents/reviewer.md`:
```markdown
---
name: reviewer
description: Thorough code review subagent — performs line-by-line analysis, spots bugs, suggests improvements.
---

You are a careful code reviewer. When asked to review code, you:

1. Read the full diff before commenting.
2. Flag bugs with severity (critical / warning / nit).
3. Suggest concrete fixes, not just observations.
```

## Complete Claude Code `plugin.json` example

```json
{
  "name": "dev-toolkit",
  "description": "Code review, commit helpers, and database tools",
  "version": "1.0.0",
  "author": { "name": "Acme Corp", "email": "team@acme.com" },
  "license": "MIT",
  "keywords": ["devtools", "code-review", "database"],
  "skills": [
    { "path": "skills/code-review", "description": "PR review checklist" },
    "skills/commit-helper"
  ],
  "agents": [
    { "path": "agents/reviewer.md" }
  ],
  "mcpServers": {
    "database": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres"],
      "env": { "DATABASE_URL": "${PLUGIN_DATA}/db.sqlite" }
    },
    "api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": { "Authorization": "Bearer ${API_TOKEN}" }
    }
  }
}
```

## Minimal Codex `plugin.json` example

```json
{
  "name": "db-tools",
  "version": "1.0.0",
  "description": "Database query and schema tools",
  "skills": "skills/",
  "mcpServers": ".mcp.json"
}
```
