---
name: skilltap
description: >
  Install, update, remove, list, toggle, adopt, move, configure, and inspect
  agent skills, plugins, and standalone MCP servers via the skilltap CLI.
  Also: troubleshoot config or state, sync project manifest + lockfile,
  manage taps for skill discovery, run doctor checks, configure security
  policy, migrate from legacy state, and any other operation on
  skilltap-managed content. Triggers on installing skills, managing plugins,
  injecting MCP servers, or working with skilltap.toml / state.json /
  config.toml.
license: MIT
---

# skilltap

skilltap is a CLI for installing and managing AI-agent skills, plugins, and MCP servers — think Homebrew for agent tooling. It installs to `.agents/skills/`, is agent-agnostic, and supports multiple sources (GitHub, any git URL, npm, local paths, taps).

## Mental model in three sentences

1. **Three artifact types**, each addressed explicitly: `skill` (a SKILL.md directory), `plugin` (a bundle of skills + MCPs + agent definitions), and `mcp` (a standalone MCP server). Every command takes the type as a subcommand: `install skill X`, `remove plugin Y`, `update mcp Z`.
2. **Two scopes**: `global` (lives under `~/.agents/`) and `project` (lives under `<project>/.agents/`). Smart default: inside a git repo → `project`; outside → `global`. Override with `--scope project|global`.
3. **One state file per scope** (`state.json`) plus an optional **project manifest + lockfile** (`skilltap.toml` + `skilltap.lock`) that gets committed to source control for team sync.

There is **no agent mode**, no `--agent` flag, and no `SKILLTAP_AGENT` env var. Output mode is auto-decided from TTY detection plus `--json`. For agent / CI / piped use, pass `--yes` (auto-accept prompts) and optionally `--json` (machine-readable events).

## Quick decision rules

| Asked to do… | Run this |
|---|---|
| Install a skill from GitHub | `skilltap install skill owner/repo` |
| Install a plugin from GitHub | `skilltap install plugin owner/repo` |
| Install one plugin from a multi-plugin repo | `skilltap install plugin owner/repo:plugin-name` |
| Install all publishable plugins from a repo | `skilltap install plugin owner/repo:*` |
| Install a tap-resolved skill by name | `skilltap install skill commit-helper` |
| Install a standalone MCP server | `skilltap install mcp npm:@scope/server` |
| Install from local dir (live link) | `skilltap install skill ./path` |
| Install from npm | `skilltap install skill npm:@scope/name[@1.2.3]` |
| Update everything | `skilltap update` |
| Update one item | `skilltap update <type> <name>` |
| Remove an item | `skilltap remove <type> <name> --yes` |
| Disable / re-enable an item | `skilltap toggle <type> <name>` |
| Disable one component of a plugin | `skilltap toggle plugin <name>:<component>` |
| Show what's installed | `skilltap status [--json]` |
| Show details on one item | `skilltap info <name> [--json]` |
| Move a skill between scopes | `skilltap move <name> --scope project\|global` |
| Bring an external dir under management | `skilltap adopt <path>` |
| Reconcile manifest ↔ lockfile ↔ state | `skilltap sync [--apply]` |
| Preview a source without installing | `skilltap try <type> <source>` |
| Check environment health | `skilltap doctor [--fix] [--json]` |
| Read / write config | `skilltap config get <key>` / `set <key> <value>` |
| Find a skill | `skilltap find <query> [--json]` |
| Migrate legacy config + state | `skilltap migrate` |

For the full command tree and examples, see [reference/installing.md](reference/installing.md) and [reference/managing.md](reference/managing.md).

## Common workflows

### Install a skill into the current project

```bash
# Inside a git repo — smart-scope picks `project`, no flag needed.
skilltap install skill commit-helper --also claude-code
```

`--also <agent>` creates the per-agent symlink (`.claude/skills/`, `.cursor/skills/`, etc.) so the named agent actually loads the skill. Repeatable. See [reference/filesystem.md](reference/filesystem.md) for the full path map.

### Install everything declared in `skilltap.toml`

```bash
git pull
skilltap sync --apply
```

`sync` reconciles the project manifest, lockfile, and on-disk state. With `--apply` it executes the diff via `install`/`remove`. See [reference/manifest.md](reference/manifest.md).

### Install a plugin and toggle one component off

```bash
skilltap install plugin corp/dev-toolkit --scope global --yes
skilltap toggle plugin dev-toolkit:test-generator   # disable just one component
```

`toggle plugin <name>:<component>` flips a single component's `active` state. `toggle plugin <name>` (no `:component`) opens a TUI scoped to that plugin's components.

### Pipe-friendly install (CI, agent, scripts)

```bash
skilltap install skill owner/repo --scope project --yes --json
```

When stdout is not a TTY, output mode auto-switches to `plain` (no spinners, no colors). Adding `--json` upgrades it to newline-delimited JSON events. Adding `--yes` auto-accepts the "do it?" confirmation. Security warnings still gate the install (see below).

## Constraints that bite

- **`--yes` does not bypass security scanning.** Scan warnings still prompt (or fail under `--strict` / `on_warn = "fail"`). The only switches that skip scanning are `--skip-scan` per invocation, `[security].scan = "none"`, or a `[security].trust` glob match for the source.
- **`--strict` and `--skip-scan` are mutually exclusive in spirit.** `--strict` = "abort on any warning" (one-shot `on_warn = "fail"`); `--skip-scan` = "don't even scan." Pick one.
- **Plugin-owned skills are not in `state.skills[]`.** They live under `state.plugins[].components[]`. `remove skill <name>` on a plugin component errors with a hint pointing at `remove plugin <name>` or `toggle plugin <name>:<component>`.
- **`--scope` is a value flag, not a boolean pair on `install`.** Use `--scope project` or `--scope global`. The boolean `--global` / `--project` pair only exists on `status` and `info` for filtering.
- **Legacy state hard-fails at load.** If a config or state file from before v2 is present, `loadConfig` exits with an error pointing at `skilltap migrate`. Run that command once on each machine to translate.
- **`skilltap.toml` corruption blocks project installs.** In TTY mode, install backs up to `skilltap.toml.bak`, resets to empty, and continues. In non-TTY mode it refuses with a hint to run `skilltap doctor --fix`.
- **Bare `skilltap` opens the TUI.** That requires a TTY. From a pipe or non-interactive context, bare invocation errors with `skilltap requires a TTY for the dashboard.` — use `skilltap status` for headless output instead.

## Reference pages

- [reference/installing.md](reference/installing.md) — `install <type> <source>`, source formats, scope resolution, `--also`, plugin capture, multi-plugin syntax, output formats.
- [reference/managing.md](reference/managing.md) — `update`, `remove`, `toggle`, `adopt`, `move`, `info`, listing via `status`, `try` (preview), `doctor`.
- [reference/non-interactive.md](reference/non-interactive.md) — output modes (tty / plain / json), exit codes, `--yes` semantics, security policy under automation, removed-command errors.
- [reference/manifest.md](reference/manifest.md) — `skilltap.toml` and `skilltap.lock` schemas, `sync` semantics, drift categories, `[[mcps]]` standalone entries.
- [reference/config.md](reference/config.md) — full `config.toml` schema (flat `[security]`, `[scanner]`, `[defaults]`, `[updates]`, `[telemetry]`, `[[taps]]`), settable keys, `config security` flags.
- [reference/state-files.md](reference/state-files.md) — `state.json` schema (`skills`, `plugins`, `mcpServers`), trust info, component union, hand-edit rules.
- [reference/filesystem.md](reference/filesystem.md) — every path skilltap touches: canonical install dirs, per-agent symlinks, MCP injection points, backup files, cache.
- [reference/troubleshooting.md](reference/troubleshooting.md) — error patterns and recovery (corrupt manifest, missing symlinks, plugin capture conflicts, security blocks, MCP injection problems).

For skill discovery (find, taps, registry): load the `skilltap-find` skill.
For authoring new skills, plugins, or taps: load the `skilltap-author` skill.
