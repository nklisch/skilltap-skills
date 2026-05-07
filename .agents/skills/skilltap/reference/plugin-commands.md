# Plugin command group

Plugins are bundles of skills, MCP servers, and agent definitions installed as a single unit. Introduced in v0.10.0; HTTP MCP server support added in v0.10.8.

## What plugins are

A plugin packages multiple components together under one installable source:
- **Skills** — SKILL.md-based instruction files, placed in `.agents/skills/{name}/`
- **MCP servers** — injected into per-agent config files with a namespaced key (`skilltap:<plugin>:<server>`). Two server types: `stdio` (command + args + optional env) and `http` (url + optional headers). The variables `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` are substituted at inject time.
- **Agent definitions** — `.md` files placed in `.claude/agents/{name}.md` (Claude Code only currently)

Two plugin formats are supported: Claude Code (`.claude-plugin/plugin.json`) and Codex (`.codex-plugin/plugin.json`).

## Install

Plugin install uses the same `skilltap install` command — no separate plugin subcommand:

```bash
skilltap install user/repo --global
skilltap install tap-name/plugin-name --project
skilltap install ./local-path --global
```

When `skilltap install` clones a repo, plugin detection runs before skill scanning:
1. Check for `.claude-plugin/plugin.json` → Claude Code plugin
2. Check for `.codex-plugin/plugin.json` → Codex plugin
3. Neither found → standard skill scanning

If a plugin manifest is detected, the install proceeds as a plugin install. In agent mode the "Install as plugin?" prompt auto-accepts. If the user explicitly declines (human mode only), the install falls back to skill-only scanning.

MCP entries are injected into all `--also` target agent configs. A `.skilltap.bak` backup is written before the first modification of any agent config file. Removal preserves user-added entries (non-`skilltap:`-namespaced).

Plugin-owned skills are tracked in `plugins.json`, NOT `installed.json`. Do not attempt to remove them individually with `skilltap skills remove`.

## Plugin state files

- Global: `~/.config/skilltap/plugins.json`
- Project: `<project>/.agents/plugins.json`

## Commands

| Command | Description |
|---------|-------------|
| `skilltap plugin [--global\|--project] [--json]` | List installed plugins |
| `skilltap plugin info <name> [--json]` | Show plugin details and components |
| `skilltap plugin toggle <name> [--skills] [--mcps] [--agents] [--json]` | Toggle component categories |
| `skilltap plugin remove <name> [--json]` | Remove plugin and all components |

Also available as `skilltap plugins` (alias for `skilltap plugin`).

## Agent-mode rules

- `plugin toggle` without `--skills`, `--mcps`, or `--agents` exits 1 with: `Provide --skills, --mcps, or --agents to specify what to toggle` — the interactive multiselect is human-mode only
- `plugin remove` auto-applies `--yes` in agent mode (no confirmation prompt)
- `--json` is preferred for parsing
- Exit codes: `0` success, `1` error

## skilltap plugin

List all installed plugins.

```bash
skilltap plugin [--global|--project] [--json]
```

Agent-mode plain text output (one line per plugin):

```
GLOBAL dev-toolkit 3 skills, 2 MCPs, 1 agent source=corp/dev-toolkit
GLOBAL db-tools 1 skill, 1 MCP source=npm:@corp/db-tools
PROJECT project-helpers 2 skills, 1 MCP source=./plugins/helpers
```

JSON output (`--json`): array of plugin records, one per plugin.

```json
[
  {
    "name": "dev-toolkit",
    "scope": "global",
    "repo": "https://github.com/corp/dev-toolkit",
    "format": "claude-code",
    "ref": "main",
    "sha": "abc123de",
    "also": ["claude-code", "cursor"],
    "installedAt": "2026-04-10",
    "updatedAt": "2026-04-10",
    "components": [...]
  }
]
```

## skilltap plugin info

Show plugin details including all components and active/inactive state.

```bash
skilltap plugin info <name> [--json]
```

JSON output is the full plugin record object (same shape as one element from `plugin --json`). Components array entries have shape: `{ type, name, active, ...serverFields }`.

## skilltap plugin toggle

Enable or disable component categories within a plugin.

```bash
skilltap plugin toggle <name> --skills [--json]
skilltap plugin toggle <name> --mcps [--json]
skilltap plugin toggle <name> --agents [--json]
```

Flags can be combined: `--skills --mcps` targets both categories.

Toggle behavior: each targeted component's active state is flipped individually (active → disabled, disabled → active). With `--skills`, every skill in the plugin is toggled regardless of its current state — there is no "set all to enabled" mode. To force a specific state for a mixed-state plugin, call `plugin info --json` first, then call `plugin toggle` only when needed.

| Component type | Enable | Disable |
|----------------|--------|---------|
| skill | Move from `.disabled/` back to `.agents/skills/`, recreate agent symlinks | Move to `.disabled/`, remove agent symlinks |
| mcp | Re-inject entry into all target agent config files | Remove entry from all agent config files |
| agent | Move from `.disabled/` back to `.claude/agents/` | Move to `.disabled/` subdirectory |

JSON output (`--json`): array of result objects per toggled component.

```json
[
  { "component": { "type": "mcp", "name": "database", "active": false }, "nowActive": false }
]
```

## skilltap plugin remove

Remove a plugin and all its components.

```bash
skilltap plugin remove <name> [--json]
```

Removes: all skill directories and agent symlinks, all MCP entries from agent config files, all agent definition files, and the record from `plugins.json`.

Agent-mode plain text output:

```
OK: Removed plugin dev-toolkit (3 skills, 2 MCPs, 1 agent)
```

JSON output (`--json`):

```json
{ "removed": "dev-toolkit", "components": "3 skills, 2 MCPs, 1 agent" }
```
