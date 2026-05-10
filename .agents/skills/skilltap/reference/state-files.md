# State Files

`state.json` is the single canonical state store. The legacy `installed.json` and `plugins.json` files (v0.x) are read **only** by `skilltap migrate`; nothing else writes to them.

## Paths

| Scope | Location |
|---|---|
| Global | `~/.config/skilltap/state.json` (or `$XDG_CONFIG_HOME/skilltap/state.json`) |
| Project | `<projectRoot>/.agents/state.json` |

Project root is determined by walking up from CWD looking for `.git` or `skilltap.toml`. If neither exists, the cwd is used.

## Top-level schema

```typescript
const StateSchema = z.object({
  version: z.literal(2),
  skills:     z.array(InstalledSkillSchema).default([]),
  plugins:    z.array(PluginRecordSchema).default([]),
  mcpServers: z.array(StoredMcpStandaloneSchema).default([]),
})
```

Three slices: `skills` (managed + linked standalone skills), `plugins` (plugins + their components), `mcpServers` (standalone MCP servers — first-class, separate from plugin-owned MCPs).

## `skills[]` — InstalledSkillSchema

```typescript
const InstalledSkillSchema = z.object({
  name:        z.string(),
  description: z.string().default(""),
  repo:        z.string().nullable(),
  ref:         z.string().nullable(),
  sha:         z.string().nullable().default(null),
  scope:       z.enum(["global", "project", "linked"]),
  path:        z.string().nullable(),
  tap:         z.string().nullable().default(null),
  also:        z.array(z.string()).default([]),
  installedAt: z.iso.datetime(),
  updatedAt:   z.iso.datetime().default("1970-01-01T00:00:00.000Z"),
  trust:       TrustInfoSchema.optional(),
  active:      z.boolean().default(true),
})
```

| Field | Type | Notes |
|---|---|---|
| `name` | string | Skill directory name. Must match the directory under `.agents/skills/`. |
| `description` | string | From SKILL.md frontmatter; default `""`. |
| `repo` | string \| null | Source repo URL or shorthand. `null` for local links. `npm:@scope/name` for npm. |
| `ref` | string \| null | Branch / tag / semver pinned at install. `null` if latest. |
| `sha` | string \| null | Commit SHA at install/update. `null` for npm and linked. |
| `scope` | `"global"` \| `"project"` \| `"linked"` | Where the skill lives. `linked` = local symlink. |
| `path` | string \| null | Absolute path to symlink target (for `scope: linked`). `null` for managed. |
| `tap` | string \| null | Tap name if installed via tap lookup. `null` otherwise. |
| `also` | string[] | Agent IDs that have a per-agent symlink. |
| `installedAt` | ISO datetime | When the skill was first installed. |
| `updatedAt` | ISO datetime | Last update time. Default `1970-01-01T00:00:00.000Z` if never updated. |
| `trust` | object? | Trust tier + provenance metadata. See below. |
| `active` | boolean | `false` when disabled. |

### Field population by source type

| Field | git install | npm install | local link |
|---|---|---|---|
| `repo` | URL or shorthand | `npm:@scope/name` | `null` |
| `ref` | branch / tag or null | semver or null | `null` |
| `sha` | commit SHA | `null` | `null` |
| `scope` | `global` or `project` | `global` or `project` | `linked` |
| `path` | `null` | `null` | absolute target path |
| `tap` | tap name or null | `null` | `null` |

## `plugins[]` — PluginRecordSchema

```typescript
const PluginRecordSchema = z.object({
  name:        z.string(),
  description: z.string().default(""),
  format:      z.enum(["claude-code", "codex", "skilltap"]),
  repo:        z.string().nullable(),
  ref:         z.string().nullable(),
  sha:         z.string().nullable(),
  scope:       z.enum(["global", "project"]),
  path:        z.string().nullable().default(null),
  also:        z.array(z.string()).default([]),
  tap:         z.string().nullable().default(null),
  components:  z.array(StoredComponentSchema),
  installedAt: z.iso.datetime(),
  updatedAt:   z.iso.datetime(),
  active:      z.boolean().default(true),
})
```

| Field | Notes |
|---|---|
| `format` | `"claude-code"` (parsed `.claude-plugin/plugin.json`), `"codex"` (parsed `.codex-plugin/plugin.json`), or `"skilltap"` (parsed `.skilltap/<name>.toml` — native format). |
| `path` | External cache path for adopted plugins (e.g. Claude Code marketplace plugins under management). |
| `components[]` | Discriminated union by `type` — see below. |
| `active` | `false` if the whole plugin is toggled off. Component-level `active` is independent. |

### Component union (`components[]`, discriminated by `type`)

#### Skill component

```json
{ "type": "skill", "name": "code-review", "active": true }
```

Plugin-owned skill. Lives in `.agents/skills/<name>/` like a standalone, but is recorded HERE (not in `state.skills[]`).

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

#### MCP server — http

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

#### Agent definition component

```json
{
  "type": "agent",
  "name": "code-review",
  "active": true,
  "platform": "claude-code"
}
```

`name` matches the file under `.claude/agents/<name>.md`. `platform` is `"claude-code"` only currently.

## `mcpServers[]` — standalone MCP servers

Standalone MCP installs (`skilltap install mcp <source>`) live here, separate from plugin-owned MCPs.

```typescript
const StoredMcpStandaloneSchema = z.object({
  name:        z.string(),
  source:      z.string(),
  config:      z.union([StoredMcpStdioConfigSchema, StoredMcpHttpConfigSchema]),
  targets:     z.array(z.string()).default([]),
  installedAt: z.iso.datetime(),
})
```

| Field | Notes |
|---|---|
| `name` | User-chosen install name. Used in the namespaced agent config key `skilltap:standalone:<name>`. |
| `source` | Original source string (`github:`, `npm:`, etc.). |
| `config` | Either stdio (`type`/`command`/`args`/`env`) or http (`type`/`url`/`headers`). |
| `targets` | Agent IDs the MCP entry was injected into (`["claude-code", "cursor"]`). |
| `installedAt` | ISO datetime. |

## `trust` — TrustInfoSchema

Stored on each `skills[]` and `plugins[]` record (optional).

```typescript
const TrustInfoSchema = z.object({
  tier: z.enum(["provenance", "publisher", "curated", "unverified"]),
  npm:    z.object({ publisher: z.string(), verifiedAt: z.string() }).optional(),
  github: z.object({ verified: z.boolean(),  repo: z.string()       }).optional(),
  tap:    z.object({ verified: z.boolean(),  verifiedBy: z.string().optional() }).optional(),
}).optional()
```

| Tier | Source of trust |
|---|---|
| `provenance` | npm Sigstore Trusted Publishing OR GitHub Artifact Attestations — cryptographically verified build chain. |
| `publisher` | Identity known (npm username, GitHub owner) but not cryptographically verified. |
| `curated` | Skill comes from a tap whose entry has `trust.verified: true` in `tap.json`. |
| `unverified` | No signal — no attestation, no known publisher, not from a verified tap. |

**Trust is informational. It never blocks an install.** Security policy (`security.scan` + `security.on_warn`) decides whether an install proceeds.

## Example state.json

```json
{
  "version": 2,
  "skills": [
    {
      "name": "commit-helper",
      "description": "Generates conventional commit messages",
      "repo": "github:nathan/commit-helper",
      "ref": "v1.2.0",
      "sha": "abc123...",
      "scope": "global",
      "path": null,
      "tap": null,
      "also": ["claude-code"],
      "installedAt": "2026-05-05T12:00:00.000Z",
      "updatedAt": "2026-05-05T12:00:00.000Z",
      "trust": { "tier": "publisher", "npm": { "publisher": "nathan", "verifiedAt": "..." } },
      "active": true
    }
  ],
  "plugins": [
    {
      "name": "dev-toolkit",
      "format": "skilltap",
      "repo": "github:corp/dev-toolkit",
      "ref":  "main",
      "sha":  "def456...",
      "scope": "global",
      "components": [
        { "type": "skill", "name": "code-review", "active": true },
        { "type": "mcp",   "serverType": "stdio", "name": "database", "active": true, "command": "...", "args": [], "env": {} },
        { "type": "agent", "name": "reviewer", "active": true, "platform": "claude-code" }
      ],
      "installedAt": "...",
      "updatedAt":   "...",
      "active": true
    }
  ],
  "mcpServers": [
    {
      "name": "search",
      "source": "github:corp/search-mcp",
      "config": { "type": "stdio", "command": "node", "args": ["./bin/search.js"], "env": {} },
      "targets": ["claude-code"],
      "installedAt": "..."
    }
  ]
}
```

## Editing state.json by hand

Supported but risky. Prefer CLI commands. If you must edit:

- Validate JSON before saving (invalid JSON causes parse failure on the next skilltap command).
- The `version` field must remain `2`.
- Removing a record from `state.skills[]` does NOT delete the skill directory — run `skilltap remove skill <name>` for proper cleanup.
- Removing a plugin record does NOT clean up MCP injections in agent config files — run `skilltap remove plugin <name>` instead.
- Run `skilltap doctor` after editing to detect inconsistencies.

## Legacy state files

If `~/.config/skilltap/installed.json` or `~/.config/skilltap/plugins.json` exists alongside no `state.json`, skilltap shows a soft hint at startup:

```
↑  Legacy state detected. Run 'skilltap migrate' to upgrade.
```

`migrate` translates and renames the originals to `*.v1.bak`. After that, only `state.json` is read; the `.v1.bak` files are kept as recovery copies but never touched again.
