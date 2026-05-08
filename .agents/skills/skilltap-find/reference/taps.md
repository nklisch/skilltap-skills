# Taps

A tap is an indexed source of installable skills and plugins. Two tap types exist:

- **git** — a git repository containing a `tap.json` at its root. Cloned locally; `tap update` pulls the latest commits.
- **http** — an HTTP endpoint serving a `tap.json` document. Never cloned; `tap update` refreshes the cached copy. Detected automatically by URL pattern, or force with `--type http`.

From an agent's perspective both types behave identically for `find` and `install`.

## Built-in tap

`skilltap-skills` is bundled with the CLI. It is automatically cloned on first use and searched by `find`. Do not add it manually with `tap add`. Toggle it with the `builtin_tap` config key:

```toml
# config.toml
builtin_tap = false   # disable
```

## Tap subcommands

```
skilltap tap add <name> <url> [--type git|http]
```
Add a tap. `--type` is optional — the CLI auto-detects git vs. HTTP. For GitHub-hosted taps, use the shorthand form instead:

```
skilltap tap add <owner/repo>
```
Derives the tap name from the repo name and the URL from `default_git_host`.

```
skilltap tap remove <name> [--yes]
```
Remove a tap. Omit `--yes` to confirm interactively.

```
skilltap tap list [--json]
```
List all configured taps. `--json` for machine-readable output.

```
skilltap tap update [name]
```
Re-fetch the tap index. Git taps: `git pull`. HTTP taps: refresh cached `tap.json`. Omit `name` to update all taps.

```
skilltap tap info <name> [--json]
```
Show metadata: name, type, URL, local path, last fetched timestamp, skill count. The `type` field is `"git"`, `"http"`, or `"builtin"`.

```
skilltap tap install [--tap <name>]      # HUMAN ONLY
skilltap tap init <name>                 # HUMAN ONLY
```
`tap install` opens a multiselect picker to install skills from a tap. `tap init` scaffolds a new tap repository. Both require an interactive TTY — do not invoke from agent mode.

## `tap.json` schema

The index file served by every tap (git or HTTP):

```json
{
  "name": "home",
  "description": "...",
  "skills": [
    {
      "name": "commit-helper",
      "description": "...",
      "repo": "https://github.com/example/commit-helper",
      "tags": ["git"],
      "trust": { "verified": true, "verifiedBy": "...", "verifiedAt": "..." },
      "plugin": false
    }
  ],
  "plugins": [
    {
      "name": "dev-toolkit",
      "description": "...",
      "version": "1.0.0",
      "skills": [
        { "name": "code-review", "path": "skills/code-review", "description": "..." }
      ],
      "mcpServers": "mcp.json",
      "agents": [{ "name": "reviewer", "path": "agents/reviewer.md" }],
      "tags": ["plugin"]
    }
  ]
}
```

Schema notes:
- `skills[].trust.verified` — when `true`, the skill gets `curated` trust tier (see `reference/trust.md`).
- `skills[].plugin` — `true` marks entries that are tap-defined plugins (displayed with `[plugin]` badge in `find` output).
- `plugins[]` — first-class plugin definitions. Install with `tap-name/plugin-name`.
- `plugins[].mcpServers` — either a path string (relative to tap root) pointing to an MCP config file, or an inline MCP server map.

## HTTP registry taps

HTTP taps serve `tap.json` over HTTPS. There is no local clone.

**Authentication** for private registries goes in `config.toml` — not via CLI flags:

```toml
[[taps]]
name = "private-registry"
url = "https://registry.example.com/tap.json"
type = "http"
auth_token = "tok_abc123"        # literal token
# OR
auth_env = "MY_REGISTRY_TOKEN"  # name of env var holding the token
```

The token is sent as a Bearer header on each `tap update` and during `find`.
