---
name: skilltap
description: >
  Install, update, remove, list, disable, enable, adopt, move, link, unlink,
  configure, and inspect agent skills and plugins via the skilltap CLI.
  Also: troubleshoot skilltap config or state, toggle plugin components
  (MCPs, agent definitions, skills), manage taps for skill discovery,
  run doctor checks, introspect installed.json or plugins.json, configure
  security policy, and any other operation on skilltap-managed skills or plugins.
license: MIT
---

# skilltap

skilltap is a CLI for installing and managing AI-agent skills and plugins — think Homebrew for agent skills. It installs to `.agents/skills/`, is agent-agnostic, and supports multiple sources (GitHub, any git URL, npm, local paths, tap registries).

## Agent-mode gate (mandatory first step)

Before any operation, check that agent mode is enabled:

```bash
skilltap status --json
```

Parse the output and check `agentMode === true`. If it is `false`, **stop**. Tell the user:

> Agent mode is not enabled. Please run `skilltap config agent-mode` in your terminal to enable it, then retry.

Do NOT proceed with any install, update, or management command until agent mode is confirmed enabled.

See [reference/agent-mode.md](reference/agent-mode.md) for the full output schema and rules.

## Quick decision rules

| Asked to do... | Run this |
|---|---|
| Install a skill or plugin by GitHub user/repo | `skilltap install user/repo [--project\|--global]` |
| Install from tap by name | `skilltap install skill-name [--project\|--global]` |
| Install from npm | `skilltap install npm:@scope/name` |
| Install a local directory | `skilltap install ./path/to/skill` |
| Update one skill | `skilltap update skill-name` |
| Update all skills | `skilltap update` |
| List all skills | `skilltap skills [--project\|--global]` |
| Get details on a skill | `skilltap skills info skill-name` |
| Disable a skill | `skilltap skills disable skill-name` |
| Enable a skill | `skilltap skills enable skill-name` |
| Remove a skill | `skilltap skills remove skill-name` |
| List plugins | `skilltap plugin [--project\|--global]` |
| Get plugin details | `skilltap plugin info plugin-name` |
| Toggle plugin components | `skilltap plugin toggle plugin-name [--skills\|--mcps\|--agents]` |
| Remove a plugin | `skilltap plugin remove plugin-name` |
| Check environment health | `skilltap doctor` |
| Read a config value | `skilltap config get key` |
| Write a config value | `skilltap config set key value` |
| Check agent mode / security status | `skilltap status --json` |

## Command reference

| Command | Purpose |
|---|---|
| `skilltap install <source>` | Install a skill or plugin (auto-detects plugin manifests) |
| `skilltap update [name]` | Update installed skills; omit name to update all |
| `skilltap skills` | List skills (managed + linked + unmanaged) |
| `skilltap skills info <name>` | Show skill record details |
| `skilltap skills remove [name...]` | Remove one or more skills |
| `skilltap skills disable <name>` | Disable a skill (moves to `.disabled/`) |
| `skilltap skills enable <name>` | Re-enable a disabled skill |
| `skilltap skills adopt [name...]` | Adopt unmanaged skills under skilltap management |
| `skilltap skills move <name>` | Move skill between scopes (`--project` / `--global`) |
| `skilltap skills link <path>` | Symlink a local skill directory |
| `skilltap skills unlink <name>` | Remove a linked skill |
| `skilltap plugin` | List installed plugins |
| `skilltap plugin info <name>` | Show plugin details and component states |
| `skilltap plugin toggle <name>` | Flip component active states (`--skills` / `--mcps` / `--agents`) |
| `skilltap plugin remove <name>` | Remove a plugin and all its components |
| `skilltap status [--json]` | Agent-mode status report |
| `skilltap doctor [--fix]` | Check environment and state; `--fix` repairs symlinks |
| `skilltap config get [key]` | Read config value (any key) |
| `skilltap config set <key> <value>` | Write config value (allowlisted keys only) |
| `skilltap config agent-mode` | Toggle agent mode (TTY-only, call from terminal) |
| `skilltap config security` | Configure security (TTY wizard or flags) |
| `skilltap config telemetry` | Status / enable / disable telemetry |
| `skilltap find [query]` | Search taps and skills.sh registry |
| `skilltap tap` | Manage tap sources |
| `skilltap create [name]` | Scaffold a new skill |
| `skilltap verify [path]` | Validate a skill |
| `skilltap self-update` | Update the CLI binary |
| `skilltap completions <shell>` | Generate shell completions |

Silent aliases (documented once — prefer canonical forms above): `list` → `skills`, `remove` → `skills remove`, `info` → `skills info`, `link` → `skills link`, `unlink` → `skills unlink`, `plugins` → `plugin`.

## Important constraints in agent mode

- `--yes` is auto-applied. Never pass it explicitly.
- `--skip-scan` is rejected by default (require_scan enforced). Do not pass it.
- `--no-strict` has no effect in agent mode.
- Scope must be `--project` or `--global`. If neither is passed, the configured default (`agent-mode.scope`, default `project`) is used.
- Plugin install prompts auto-accept. Multi-skill repos auto-select all skills.
- Security warnings cause exit 1 with a `SECURITY ISSUE FOUND` block — relay verbatim to user, do not retry.

## Reference pages

- [reference/installing.md](reference/installing.md) — Source formats, install decision tree, plugin auto-detection, output formats
- [reference/managing.md](reference/managing.md) — Post-install skill and plugin management (list, disable, enable, remove, adopt, move, link, plugin commands)
- [reference/agent-mode.md](reference/agent-mode.md) — Status check schema, flag rules, exit codes, output formats, security block handling
- [reference/config.md](reference/config.md) — Full config.toml schema, settable vs blocked keys, config subcommands
- [reference/state-files.md](reference/state-files.md) — installed.json and plugins.json schemas, field types, component union
- [reference/filesystem.md](reference/filesystem.md) — Where everything lives on disk, globalBase resolution, symlink semantics
- [reference/troubleshooting.md](reference/troubleshooting.md) — Error patterns and recovery steps

For skill discovery (find, tap management): load the `skilltap-find` skill.
For authoring new skills or plugins: load the `skilltap-author` skill.
