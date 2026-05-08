# State Files

## installed.json

Tracks managed and linked skills. Plugin-owned skills are NOT in this file — see `plugins.json`.

### Paths

| Scope | Path |
|---|---|
| Global | `~/.config/skilltap/installed.json` |
| Project | `<project-root>/.agents/installed.json` |

### Schema

```json
{
  "version": 1,
  "skills": [
    {
      "name": "<skill-name>",
      "description": "",
      "repo": "owner/repo or https://... or null",
      "ref": "v1.2.0 or branch-name or null",
      "sha": "abc123def456... or null",
      "scope": "global | project | linked",
      "path": "<absolute path> or null",
      "tap": "<tap-name> or null",
      "also": ["claude-code", "cursor"],
      "installedAt": "2025-03-15T12:00:00.000Z",
      "updatedAt": "1970-01-01T00:00:00.000Z",
      "trust": {
        "tier": "provenance | publisher | curated | unverified"
      },
      "active": true
    }
  ]
}
```

### Field reference

| Field | Type | Notes |
|---|---|---|
| `version` | number | Always `1`. |
| `name` | string | Skill directory name. Matches the directory under `.agents/skills/`. |
| `description` | string | From SKILL.md frontmatter; default `""`. |
| `repo` | string \| null | Source repo URL or shorthand. null for local links. `npm:@scope/name` for npm. |
| `ref` | string \| null | Branch, tag, or semver pinned at install. null if latest. |
| `sha` | string \| null | Commit SHA at install/update time. null for npm and linked. |
| `scope` | `"global"` \| `"project"` \| `"linked"` | Where the skill lives. `linked` = local symlink. |
| `path` | string \| null | Absolute path to symlink target (for `scope: linked`). null for managed. |
| `tap` | string \| null | Tap name if installed via tap lookup. null otherwise. |
| `also` | string[] | Agent IDs that have a symlink into their skills dir. |
| `installedAt` | ISO datetime string | When the skill was first installed. |
| `updatedAt` | ISO datetime string | Last update time. Default `1970-01-01T00:00:00.000Z` if never updated. |
| `trust.tier` | string | One of `provenance`, `publisher`, `curated`, `unverified`. Informational only. |
| `active` | boolean | `false` when the skill is disabled. |

### Which fields are set per source type

| Field | git install | npm install | local link |
|---|---|---|---|
| `repo` | URL or shorthand | `npm:@scope/name` | null |
| `ref` | branch/tag or null | semver or null | null |
| `sha` | commit SHA | null | null |
| `scope` | `global` or `project` | `global` or `project` | `linked` |
| `path` | null | null | absolute target path |
| `tap` | tap name or null | null | null |

## plugins.json

Tracks installed plugins and their components.

### Paths

| Scope | Path |
|---|---|
| Global | `~/.config/skilltap/plugins.json` |
| Project | `<project-root>/.agents/plugins.json` |

### Schema

```json
{
  "version": 1,
  "plugins": [
    {
      "name": "dev-toolkit",
      "description": "",
      "format": "claude-code | codex | skilltap",
      "repo": "https://... or null",
      "ref": "v1.0.0 or null",
      "sha": "abc123... or null",
      "scope": "global | project",
      "also": ["claude-code", "cursor"],
      "tap": "<tap-name> or null",
      "components": [...],
      "installedAt": "2025-03-15T12:00:00.000Z",
      "updatedAt": "1970-01-01T00:00:00.000Z",
      "active": true
    }
  ]
}
```

### Plugin-level fields

| Field | Type | Notes |
|---|---|---|
| `name` | string | Plugin identifier. |
| `description` | string | From plugin manifest; default `""`. |
| `format` | string | `claude-code`, `codex`, or `skilltap` — which manifest format was detected. |
| `repo` | string \| null | Source repo URL. |
| `ref` | string \| null | Branch or tag at install. |
| `sha` | string \| null | Commit SHA at install. |
| `scope` | `"global"` \| `"project"` | Where the plugin was installed. |
| `also` | string[] | Agent IDs with symlinks. |
| `tap` | string \| null | Tap name if installed via tap. |
| `installedAt` | ISO datetime string | Install timestamp. |
| `updatedAt` | ISO datetime string | Last update. Default `1970-01-01T00:00:00.000Z`. |
| `active` | boolean | `false` if the whole plugin is toggled off. |

### Component union (discriminated by `type`)

#### Skill component

```json
{ "type": "skill", "name": "code-review", "active": true }
```

| Field | Type |
|---|---|
| `type` | `"skill"` |
| `name` | string — must match a directory under `.agents/skills/` |
| `active` | boolean |

#### MCP server — stdio

```json
{
  "type": "mcp",
  "serverType": "stdio",
  "name": "database",
  "active": true,
  "command": "npx",
  "args": ["-y", "mcp-db"],
  "env": { "DB_URL": "..." }
}
```

| Field | Type |
|---|---|
| `type` | `"mcp"` |
| `serverType` | `"stdio"` |
| `name` | string — used in namespaced key `skilltap:<plugin>:<name>` |
| `active` | boolean |
| `command` | string |
| `args` | string[] |
| `env` | Record<string, string> |

#### MCP server — http (added v0.10.8)

```json
{
  "type": "mcp",
  "serverType": "http",
  "name": "api",
  "active": true,
  "url": "https://mcp.example.com/sse",
  "headers": { "Authorization": "Bearer ..." }
}
```

| Field | Type |
|---|---|
| `type` | `"mcp"` |
| `serverType` | `"http"` |
| `name` | string |
| `active` | boolean |
| `url` | string |
| `headers` | Record<string, string> |

#### Agent definition component

```json
{
  "type": "agent",
  "name": "code-review",
  "active": true,
  "platform": "claude-code"
}
```

| Field | Type |
|---|---|
| `type` | `"agent"` |
| `name` | string — matches file under `.claude/agents/` |
| `active` | boolean |
| `platform` | string — currently `"claude-code"` only |

## Editing state files by hand

Supported but risky. Prefer CLI commands. If you must edit:
- Validate JSON before saving (invalid JSON causes parse failure on next skilltap command).
- Run `skilltap doctor` after editing to detect inconsistencies.
- The `version` field must remain `1`.
- Removing a record from `installed.json` does NOT delete the skill directory — run `skilltap skills remove` for proper cleanup.
