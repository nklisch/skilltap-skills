# Tap Format Reference

A tap is a curated index of skills and plugins — a `tap.json` file in a git repository. Think of it as a registry that maps short names to source repos. Users add a tap once (`skilltap tap add <name> <url>`) and then install from it by name (`skilltap install <skill-name>`).

Taps can also embed complete plugin definitions inline, so a tap maintainer can ship everything in one file without authors needing separate plugin manifest repos.

## Scaffold a tap

```bash
skilltap tap init <name>
```

Creates:
```
<name>/
├── tap.json    (skeleton — edit this)
└── .git/       (initialized)
```

Set a remote and push. Users add it with:
```bash
skilltap tap add <name> https://github.com/you/<name>
```

## `tap.json` schema

Source of truth: `packages/core/src/schemas/tap.ts`.

### Top-level fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | Tap identifier |
| `description` | string | no | One-line summary |
| `skills` | `TapSkill[]` | yes | Listed skills (may be empty array) |
| `plugins` | `TapPlugin[]` | no | Inline plugin definitions (default `[]`) |

### `TapSkill` fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | Skill name (must match `frontmatter.name` in the skill) |
| `description` | string | yes | One-line description shown in `skilltap find` |
| `repo` | string | yes | `owner/repo` shorthand or full URL |
| `tags` | string[] | no | Category tags (default `[]`) |
| `trust` | `TapTrust` | no | Verification metadata |
| `plugin` | boolean | no | `true` if this entry is a plugin, not a skill (default `false`) |

### `TapTrust` fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `verified` | boolean | yes | Whether the tap maintainer has vetted this entry |
| `verifiedBy` | string | no | Who curated it (name or handle) |
| `verifiedAt` | string | no | ISO 8601 datetime of last verification |

When `verified: true`, the skill gets the `curated` trust tier on install. See [../skilltap-find/](../skilltap-find/) for trust tier details.

### `TapPlugin` fields (inline plugin definitions)

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `name` | string | yes | Plugin name |
| `description` | string | no | One-liner (default `""`) |
| `version` | string | no | SemVer |
| `skills` | `TapPluginSkill[]` | no | Skill entries (default `[]`) |
| `mcpServers` | string \| object | no | Path string OR inline server map |
| `agents` | `TapPluginAgent[]` | no | Agent definition entries (default `[]`) |
| `tags` | string[] | no | Category tags (default `[]`) |

### `TapPluginSkill` fields

| Field | Type | Required |
|-------|------|----------|
| `name` | string | yes |
| `path` | string | yes | Path relative to the plugin's install root |
| `description` | string | no (default `""`) |

### `TapPluginAgent` fields

| Field | Type | Required |
|-------|------|----------|
| `name` | string | yes |
| `path` | string | yes | Path to the `.md` file, relative to install root |

## HTTP registry option

skilltap also supports HTTP registries as taps — a hosted endpoint that serves the same `tap.json` schema over HTTPS. Point users at the registry URL instead of a git repo. Refer to the skilltap docs for the HTTP registry API spec; it is not reproduced here.

## Complete `tap.json` example

```json
{
  "name": "acme-tap",
  "description": "Curated skills and plugins from Acme Corp",
  "skills": [
    {
      "name": "code-review",
      "description": "PR review checklist and style guide enforcer",
      "repo": "acme-corp/code-review-skill",
      "tags": ["devtools", "code-quality"],
      "trust": {
        "verified": true,
        "verifiedBy": "acme-tap-maintainers",
        "verifiedAt": "2025-01-15T00:00:00Z"
      }
    },
    {
      "name": "db-toolkit",
      "description": "Database tools plugin with MCP server",
      "repo": "acme-corp/db-toolkit",
      "tags": ["database", "sql"],
      "plugin": true,
      "trust": {
        "verified": true,
        "verifiedBy": "acme-tap-maintainers",
        "verifiedAt": "2025-02-01T00:00:00Z"
      }
    },
    {
      "name": "community-linter",
      "description": "Community-contributed linting helpers",
      "repo": "someuser/linter-skill",
      "tags": ["linting"],
      "trust": {
        "verified": false
      }
    }
  ],
  "plugins": [
    {
      "name": "internal-tools",
      "description": "Internal Acme developer tools",
      "version": "2.1.0",
      "tags": ["internal", "devtools"],
      "skills": [
        {
          "name": "deploy-helper",
          "path": "skills/deploy-helper",
          "description": "Guided deployment workflow"
        },
        {
          "name": "incident-runbook",
          "path": "skills/incident-runbook",
          "description": "On-call incident response steps"
        }
      ],
      "mcpServers": {
        "ci": {
          "type": "http",
          "url": "https://ci.acme.internal/mcp"
        },
        "vault": {
          "type": "stdio",
          "command": "npx",
          "args": ["-y", "@acme/vault-mcp"]
        }
      },
      "agents": [
        {
          "name": "deploy-agent",
          "path": "agents/deploy-agent.md"
        }
      ]
    }
  ]
}
```

## Maintenance workflow

1. Fork/clone your tap repo.
2. Add or update entries in `tap.json`.
3. Set `trust.verified` and `trust.verifiedAt` after vetting each entry.
4. Commit and push — consumers get updates next time they run `skilltap tap update`.

To accept community submissions, open PRs against your tap repo. Review the linked skill repo, set `trust.verified: true` if satisfied, merge.
