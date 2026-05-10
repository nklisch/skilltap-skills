# Plugin Format Reference

A plugin bundles skills, MCP servers, and agent definitions into a single installable unit. When a user runs `skilltap install plugin <source>`, skilltap detects the format, parses the manifest, copies skills into `.agents/skills/`, injects MCP servers into per-agent config files, and places agent definitions where the agent expects them.

## Detection order

| Priority | Format | Manifest path |
|---|---|---|
| 1 | **skilltap (native)** | `.skilltap/<name>.toml` |
| 2 | Claude Code | `.claude-plugin/plugin.json` |
| 3 | Codex | `.codex-plugin/plugin.json` |

Detection runs in that order. If none found, `install plugin` errors with a hint to use `install skill`. All three formats normalize to the same internal `PluginManifest`.

For new plugins, prefer the **native skilltap format** — it is multi-plugin friendly, supports stdio + http MCP servers and Claude Code agent definitions, and uses an explicit `publish = true` opt-in.

## Native skilltap format (`.skilltap/<name>.toml`)

A repo opts in by adding one or more files under `.skilltap/<plugin-name>.toml`. Each file is independently publishable.

```toml
# .skilltap/team-toolkit.toml

name        = "team-toolkit"
version     = "1.0.0"
description = "Internal dev tools"
publish     = true                  # required, default false; explicit opt-in

[[skills]]
name = "code-review"
path = "./skills/code-review"

[[skills]]
name = "lint-checker"
path = "./skills/lint-checker"

[[servers]]                         # MCP servers
name    = "db"
type    = "stdio"                   # "stdio" | "http"
command = "node"
args    = ["./mcp/db.js"]
env     = { DATABASE_URL = "${DATABASE_URL}" }

[[servers]]
name    = "search"
type    = "http"
url     = "https://search.internal.corp/mcp"
headers = { Authorization = "Bearer ${SEARCH_TOKEN}" }

[[agents]]
name = "reviewer"
path = "./agents/reviewer.md"
```

### Top-level fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Plugin identifier. Must match the filename stem. |
| `version` | string | no | SemVer recommended. |
| `description` | string | no | One-line summary. |
| `publish` | boolean | no (default `false`) | **Required `true` to expose this plugin to outside installers.** `false` makes the manifest project-internal — the repo can still be installed for its `[skills]` / `[plugins]` deps in `skilltap.toml`, but the plugin is not externally installable. |
| `[[skills]]` | array | no | Each skill needs `name` and `path` (relative to plugin root). |
| `[[servers]]` | array | no | MCP servers — see below. |
| `[[agents]]` | array | no | Agent definitions — `name` and `path` to a `.md` file. |

### Multi-plugin in one repo

Drop multiple files into `.skilltap/`:

```
.skilltap/
  frontend-toolkit.toml      publish = true
  backend-toolkit.toml       publish = true
  internal-only-tools.toml   publish = false
```

Install:
```bash
skilltap install plugin you/repo:frontend-toolkit   # one named plugin
skilltap install plugin you/repo:*                  # every plugin with publish = true
```

Repos with exactly one publishable plugin can be installed without the `:name` selector; repos with multiple require either `:name` or `:*` (or a TTY picker).

## Claude Code `plugin.json`

Source: `packages/core/src/schemas/plugin.ts` (`ClaudePluginJsonSchema`).

**Required:** `name`

**Optional:**

| Field | Type | Notes |
|---|---|---|
| `description` | string | One-line summary |
| `version` | string | SemVer recommended |
| `author` | `{ name, email?, url? }` | Author info |
| `homepage` | string | URL |
| `repository` | string | URL |
| `license` | string | SPDX identifier |
| `keywords` | string[] | Searchable tags |
| `skills` | string \| string[] \| `{ path, description? }`[] | Skill paths relative to plugin root. Omit → auto-scans `skills/*/SKILL.md` |
| `agents` | string \| string[] \| `{ path, name? }`[] | Agent `.md` file paths. Omit → auto-scans `agents/*.md` |
| `mcpServers` | string \| string[] \| `{ name: server }` | Path(s) to `.mcp.json` or inline server map. Omit → checks `.mcp.json` at plugin root |

**Ignored** (platform-specific, not portable): `hooks`, `lspServers`, `commands`, `outputStyles`, `channels`, `userConfig`.

Unknown fields pass through (the schema uses `.passthrough()`).

## Codex `plugin.json`

Source: `packages/core/src/schemas/plugin.ts` (`CodexPluginJsonSchema`).

**Required:** `name`, `version`, `description`

**Optional:** same shape as Claude Code except:
- `skills` — string (single path) only, not array or object form.
- `mcpServers` — string (path to `.mcp.json`) only, not inline object.
- No `agents` field — Codex does not consume agent definitions.
- `apps`, `interface` — present but ignored.

## MCP server schemas

All three formats support `stdio` and `http` server types. The `type` field defaults to `"stdio"` if omitted.

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

### HTTP server

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

Inline in `plugin.json` (Claude Code only) or in `.skilltap/<name>.toml` (`[[servers]]` arrays):

```json
{
  "mcpServers": {
    "database": { "command": "npx", "args": ["-y", "@mcp/postgres"] },
    "api":      { "type": "http", "url": "https://api.example.com/mcp" }
  }
}
```

File-based (Claude Code + Codex) — path relative to plugin root:
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
|---|---|
| `${CLAUDE_PLUGIN_ROOT}` | Absolute path to the plugin install root. |
| `${CLAUDE_PLUGIN_DATA}` | Plugin data directory (`~/.config/skilltap/plugin-data/<plugin-name>/`). |

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

Standalone MCPs (installed with `skilltap install mcp <source>`, NOT bundled in a plugin) use:

```
skilltap:standalone:<install-name>
```

On removal or toggle-off, only entries matching the appropriate `skilltap:` prefix are removed. User-added entries (e.g. `my-own-mcp`) are never touched.

## Agent definition files

Place agent definitions at `agents/<name>.md` in the plugin root, or specify explicit paths in the manifest (`agents` field for Claude Code; `[[agents]]` for native).

Format: same SKILL.md frontmatter (`---`...`---`) plus a markdown body. The `name` field is optional and defaults to the filename stem.

On install, Claude Code agents are placed at `~/.claude/agents/<name>.md` (or `<project>/.claude/agents/<name>.md` for project scope). Agent definitions are file COPIES, not symlinks.

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

## Plugin capture (when authoring with already-installed standalones)

When a user installs a plugin whose components share a name (and canonical source) with already-installed standalone skills or MCPs, skilltap offers to **capture** them into the plugin record. As the plugin author, you don't need to do anything special — capture is detected automatically based on the canonical source URL match.

If your plugin's components diverge from existing standalones the user has installed (e.g. you renamed something, or your component has a different code base), users may see a cross-source conflict. Document this in your README so users know to pass `--force-capture` (replace standalones) or `--no-capture` (install side-by-side).

## Validation

Run `skilltap doctor plugin <path>` to validate:

- The chosen manifest exists and parses cleanly.
- `name` matches the directory (or `.skilltap/` filename stem).
- Every referenced skill / MCP / agent path resolves.
- Static security scan clean across all referenced files.

## Complete examples

### Native skilltap (recommended)

```toml
# .skilltap/dev-toolkit.toml
name        = "dev-toolkit"
version     = "1.0.0"
description = "Code review, commit helpers, and database tools"
publish     = true

[[skills]]
name = "code-review"
path = "./skills/code-review"

[[skills]]
name = "commit-helper"
path = "./skills/commit-helper"

[[servers]]
name    = "database"
type    = "stdio"
command = "npx"
args    = ["-y", "@modelcontextprotocol/server-postgres"]
env     = { DATABASE_URL = "${CLAUDE_PLUGIN_DATA}/db.sqlite" }

[[servers]]
name    = "api"
type    = "http"
url     = "https://api.example.com/mcp"
headers = { Authorization = "Bearer ${API_TOKEN}" }

[[agents]]
name = "reviewer"
path = "./agents/reviewer.md"
```

### Claude Code `plugin.json`

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
      "env": { "DATABASE_URL": "${CLAUDE_PLUGIN_DATA}/db.sqlite" }
    },
    "api": {
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": { "Authorization": "Bearer ${API_TOKEN}" }
    }
  }
}
```

### Minimal Codex `plugin.json`

```json
{
  "name": "db-tools",
  "version": "1.0.0",
  "description": "Database query and schema tools",
  "skills": "skills/",
  "mcpServers": ".mcp.json"
}
```
