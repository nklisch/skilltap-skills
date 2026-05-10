# Tap Format Reference

A tap is a curated index of skills and plugins — a `tap.json` file at the root of a git repository. Think of it as a registry that maps short names to source repos. Users add a tap once (`skilltap tap add <name> <url>` or `skilltap tap add owner/repo`) and then install entries from it by name.

Taps can also embed complete plugin definitions inline, so a tap maintainer can ship everything in one file without authors needing separate plugin manifest repos.

**Taps are git-only.** HTTP registry taps were removed in v2.

## Scaffold a tap

```bash
skilltap tap init my-tap
```

Creates:
```
my-tap/
├── tap.json    (skeleton — edit this)
└── .git/       (initialized)
```

Set a remote and push. Users add it with:
```bash
skilltap tap add my-tap https://github.com/you/my-tap
# or
skilltap tap add you/my-tap
```

## `tap.json` schema

Source of truth: `packages/core/src/schemas/tap.ts`.

### Top-level fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Tap identifier. |
| `description` | string | no | One-line summary. |
| `skills` | `TapSkill[]` | yes | Listed skills (may be empty). |
| `plugins` | `TapPlugin[]` | no | Inline plugin definitions (default `[]`). |

### `TapSkill` fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Skill name (must match `frontmatter.name` in the actual skill). |
| `description` | string | yes | One-line description shown in `skilltap find`. |
| `repo` | string | yes | `owner/repo` shorthand, full git URL, or `npm:@scope/name`. |
| `tags` | string[] | no | Category tags (default `[]`). |
| `trust` | `TapTrust` | no | Verification metadata. |
| `plugin` | boolean | no | `true` if this entry points at a plugin repo (vs a single-skill repo). Default `false`. |

### `TapTrust` fields

| Field | Type | Required | Notes |
|---|---|---|---|
| `verified` | boolean | yes | Whether the tap maintainer has vetted this entry. |
| `verifiedBy` | string | no | Who curated it (name or handle). |
| `verifiedAt` | string | no | ISO 8601 datetime of last verification. |

When `verified: true`, the skill gets the `curated` trust tier on install. See [../skilltap-find/reference/trust.md](../../skilltap-find/reference/trust.md) for trust tier details.

### `TapPlugin` fields (inline plugin definitions)

When you want the entire plugin definition to live inside the tap (instead of in a separate plugin repo), use `plugins[]`. Users install with `tap-name/plugin-name`.

| Field | Type | Required | Notes |
|---|---|---|---|
| `name` | string | yes | Plugin name. |
| `description` | string | no (default `""`) | One-liner. |
| `version` | string | no | SemVer. |
| `skills` | `TapPluginSkill[]` | no (default `[]`) | Skill entries. |
| `mcpServers` | string \| object | no | Path string OR inline server map. |
| `agents` | `TapPluginAgent[]` | no (default `[]`) | Agent definition entries. |
| `tags` | string[] | no (default `[]`) | Category tags. |

### `TapPluginSkill` fields

| Field | Type | Required |
|---|---|---|
| `name` | string | yes |
| `path` | string | yes — relative to the plugin's install root |
| `description` | string | no (default `""`) |

### `TapPluginAgent` fields

| Field | Type | Required |
|---|---|---|
| `name` | string | yes |
| `path` | string | yes — path to the `.md` file, relative to install root |

## Marketplace fallback

If you publish a Claude Code marketplace (`.claude-plugin/marketplace.json`) instead of a `tap.json`, skilltap will also adapt that to the internal `Tap` type when users add the repo as a tap. See [../../skilltap-find/reference/taps.md](../../skilltap-find/reference/taps.md) for the source-mapping table.

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
      "trust": { "verified": false }
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
        { "name": "deploy-agent", "path": "agents/deploy-agent.md" }
      ]
    }
  ]
}
```

## Maintenance workflow

1. Clone the tap repo.
2. Add or update entries in `tap.json`.
3. Set `trust.verified` and `trust.verifiedAt` after vetting each entry.
4. Commit and push — consumers get updates on next `find` or `install` (taps are re-fetched on use when stale).

To accept community submissions, open PRs against your tap repo. Review the linked skill repo, set `trust.verified: true` if satisfied, merge.

## Distribution tip

Once your tap is hosted, encourage users to add it via the GitHub shorthand form, which derives both name and URL:

```bash
skilltap tap add you/my-tap
```

Mention this in your repo README so the install command is one line.
