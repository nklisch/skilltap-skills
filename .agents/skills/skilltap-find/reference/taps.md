# Taps

A tap is an indexed, curated source of installable skills and plugins — a `tap.json` file at the root of a git repository. Users add a tap once (`skilltap tap add`) and then install entries from it by name.

**Taps are git-only.** HTTP registry taps (with `auth_token` / `auth_env` fields) were removed in v2. If you have a v0.x config with `[[taps]] type = "http"`, `skilltap migrate` lists them and aborts so you can convert or remove them by hand.

From an agent's perspective, every tap behaves identically for `find` and `install`.

## Built-in tap

The `skilltap-skills` tap is bundled with the CLI. It is automatically cloned on first use (lazily, to `~/.config/skilltap/taps/skilltap-skills/`) and searched by `find`. Do not add it manually with `tap add`.

Toggle it via the top-level `builtin_tap` config key:

```toml
# ~/.config/skilltap/config.toml
builtin_tap = false   # disable
```

When disabled, `find` won't search it and the lazy clone is skipped.

## Tap subcommands

```bash
skilltap tap add <name> <url>
```

Add a tap. The URL is treated as git only — there is no `--type` flag and no HTTP fallback. For GitHub-hosted taps, the shorthand form derives both name and URL:

```bash
skilltap tap add owner/repo
```

```bash
skilltap tap remove <name> [--yes]
```

Remove a tap. Omit `--yes` to confirm interactively (TTY only).

```bash
skilltap tap list [--json]
```

List all configured taps.

```bash
skilltap tap info <name> [--json]
```

Show metadata: name, type (`"git"` or `"builtin"`), URL, local path, last fetched timestamp, skill count.

```bash
skilltap tap init <directory>
```

Scaffold a new tap repo with a skeleton `tap.json` and an initialized `.git/`. Push to a remote so users can install with `skilltap tap add owner/repo`.

There is **no `tap update` subcommand** in v2.x. Taps are refreshed on use; `skilltap doctor` reports unreachable taps.

## `tap.json` schema

The index file at the root of every tap:

```json
{
  "name": "home",
  "description": "...",
  "skills": [
    {
      "name": "commit-helper",
      "description": "...",
      "repo": "owner/commit-helper",
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
- `skills[].repo` — git URL or `npm:@scope/name`. The skill is fetched from there at install time.
- `skills[].trust.verified` — when `true`, installs from this entry get the `curated` trust tier.
- `skills[].plugin` — `true` marks entries that point at a plugin repo (vs a single-skill repo). Displayed with a `[plugin]` badge in `find` output.
- `plugins[]` — first-class **inline** plugin definitions (the entire plugin lives in the tap, not in a separate repo). Install with `tap-name/plugin-name`.
- `plugins[].mcpServers` — either a path string (relative to tap root) pointing to an MCP config file, or an inline MCP server map.

Validated at clone/update with `TapSchema` (Zod 4). Invalid taps fail with a clear parse error.

## marketplace.json fallback

If a tap repo has no `tap.json` at the root, skilltap looks for `.claude-plugin/marketplace.json` (Claude Code marketplace format) and adapts it to the internal `Tap` type. This lets Claude Code marketplaces work as taps.

Source mapping (`adaptMarketplaceToTap`):

| marketplace source type | Maps to |
|---|---|
| Relative path string (no plugin.json) | `TapSkill` — uses the marketplace repo's git URL |
| Relative path string (with plugin.json) | `TapPlugin` — components from the manifest |
| `github` source | `TapSkill` — `repo` field |
| `url` source | `TapSkill` — `url` field |
| `git-subdir` source | `TapSkill` — `url` field (path not preserved) |
| `npm` source | `TapSkill` — `"npm:<package>"` |

Plugin-only fields (LSP servers, hooks, slash commands, output styles) are silently ignored. Extra fields are stripped by Zod.

## Maintenance workflow

1. Clone the tap repo.
2. Add or update entries in `tap.json`.
3. Set `trust.verified` and `trust.verifiedAt` after vetting each entry.
4. Commit and push. Consumers get updates on next `find` or `install` (taps are re-fetched on use when stale).

To accept community submissions, open PRs against the tap repo. Review the linked skill repo, set `trust.verified: true` if satisfied, merge.

## Authentication for private taps

Taps are pulled with `git`. For private repos, configure git auth normally — SSH keys, an HTTPS credential helper, or `git config` credentials. Skilltap shells out to `git` directly and inherits the user's auth (no library, no token in config).
