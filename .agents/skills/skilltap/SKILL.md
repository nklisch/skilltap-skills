---
name: skilltap
description: Manage agent skills and plugins with the skilltap CLI — install, update, remove, disable, enable, adopt, and move skills from any git host or npm; manage plugin bundles (skills + MCP servers + agent definitions). Use when the user wants to install, update, remove, list, adopt, move, disable, or enable agent skills or plugins, or configure skilltap settings.
license: MIT
---
# skilltap

Agent-mode CLI for installing, updating, removing, and managing agent skills and plugins. All output is plain text, no interactive prompts, security warnings hard-fail.

## Before using any command

Run `skilltap status` first. If `agent-mode` is disabled, **stop** and tell the user:

> "skilltap agent mode is not enabled. Please run `skilltap config agent-mode` to configure it, then try again."

Do not attempt any skill or plugin operations until agent mode is confirmed enabled.

If the user wants to understand what skilltap is, how it works, or needs help with human-mode commands (interactive wizards, config setup), see [reference/human-guide.md](reference/human-guide.md).

## Agent-mode behavior

- `--yes` is auto-applied — never pass it
- `--strict` is forced — never pass it
- `--no-strict` has no effect in agent mode — security warnings always hard-fail
- `--skip-scan` is rejected (exit 1) when `require_scan` is enabled
- Scope must be set via `--project` / `--global` or pre-configured in `agent-mode.scope`
- Security warnings always hard-fail (exit 1) — no override available
- All output is plain text to stdout; errors to stderr
- All skill names in multi-skill repos are installed automatically
- Plugin install prompts ("Install as plugin?") auto-accept in agent mode

## Command reference

| Task | Command |
|------|---------|
| Check status | `skilltap status [--json]` |
| Install skill or plugin | `skilltap install <source> [--project\|--global]` |
| Update skills | `skilltap update [name] [--check] [--json]` |
| Remove skill | `skilltap skills remove <name>` |
| List skills | `skilltap skills [--json]` |
| Skill info | `skilltap skills info <name>` |
| Disable skill | `skilltap skills disable <name>` |
| Enable skill | `skilltap skills enable <name>` |
| Adopt skill | `skilltap skills adopt [name...] [--yes]` |
| Move skill | `skilltap skills move <name> --global\|--project` |
| Find skills | `skilltap find [query] [--json]` |
| List plugins | `skilltap plugin [--json]` |
| Plugin info | `skilltap plugin info <name> [--json]` |
| Toggle plugin components | `skilltap plugin toggle <name> --skills\|--mcps\|--agents [--json]` |
| Remove plugin | `skilltap plugin remove <name> [--json]` |

Human-mode only (agents may encounter these but cannot invoke them interactively):
- `skilltap doctor` — environment check
- `skilltap verify [path]` — validate a skill before sharing
- `skilltap completions <shell>` — generate shell completions
- `skilltap create [name]` — scaffold a new skill

## Aliases

Silent aliases for backwards compatibility — prefer the canonical forms above:

| Alias | Canonical |
|-------|-----------|
| `skilltap list` | `skilltap skills` |
| `skilltap remove` | `skilltap skills remove` |
| `skilltap info` | `skilltap skills info` |
| `skilltap link` | `skilltap skills link` |
| `skilltap unlink` | `skilltap skills unlink` |
| `skilltap plugins` | `skilltap plugin` |

## Status

```bash
skilltap status --json
```

Plain text fields (one per line):
- `agent-mode: enabled|disabled`
- `scope: project|global|(not configured)`
- `security.human: <description>`
- `security.agent: <description>`
- `agent_cli: <path>|(none)`
- `also: <a> <b>|(none)`
- `taps: <count>`
- `plugins: <count>`

JSON fields: `agentMode` (boolean), `scope` (string|null), `security` (`{ human, agent, agent_cli }`), `also` (array), `taps` (number), `plugins` (number).

Always check `agentMode` is `true` before proceeding.

## Install

```bash
skilltap install <source> --project
skilltap install <source> --global
skilltap install <source> --also claude-code --also cursor
skilltap install <source> --ref v1.2.0
skilltap install <source> --semantic
skilltap install <source> --quiet
```

Source formats:
- `user/repo` — GitHub shorthand
- `github:user/repo` — GitHub explicit
- Any git URL or SSH URL
- `skill-name` — tap lookup
- `skill-name@v1.2.0` — tap lookup with version
- `tap-name/plugin-name` — tap-defined plugin
- `./local-path` — local path
- `npm:@scope/pkg` — npm registry
- `npm:@scope/pkg@1.2.3` — npm pinned version

Plugin auto-detection: if the repo contains `.claude-plugin/plugin.json` or `.codex-plugin/plugin.json`, the install is automatically treated as a plugin install. In agent mode the "Install as plugin?" prompt auto-accepts. See [reference/plugin-commands.md](reference/plugin-commands.md).

## Update

```bash
skilltap update                       # all installed skills
skilltap update <name>                # single skill
skilltap update --check               # check for updates, don't apply
skilltap update --semantic            # with semantic security scan
skilltap update --json                # JSON output
```

`--check` / `-c`: checks for available updates, refreshes the cache, prints which skills have updates — does not apply them.

Warnings on update skip the affected skill and continue. Disabled skills are skipped.

## Skills command group

Unified skill management — list, adopt, move, disable, enable, remove, info. See [reference/skills-commands.md](reference/skills-commands.md) for full command details and flags.

Note: `skilltap skills` lists skills only. Installed plugins appear via `skilltap plugin`. Plugin-owned skills do NOT appear in `installed.json` — never attempt to remove them with `skilltap skills remove`.

## Plugin command group

Plugins are bundles of skills + MCP servers + agent definitions installed as a single unit. See [reference/plugin-commands.md](reference/plugin-commands.md) for full command details, output formats, and agent-mode rules.

## Security configuration

Per-mode settings (human vs agent), presets (none/relaxed/standard/strict), and trust overrides. See [reference/security-config.md](reference/security-config.md) for configuration details.

## Taps

```bash
skilltap tap add <name> <url>
skilltap tap remove <name>
skilltap tap list
skilltap tap update [name]
```

## Output parsing

Skill install — stdout, one line per skill:
- `OK: Installed <name> -> <path>`
- `OK: <name> is already up to date.`
- `SKIP: <name> is linked.`
- `SKIP: <name> — disabled`

Plugin install emits component lines followed by a summary line. See [reference/plugin-commands.md](reference/plugin-commands.md) for plugin output formats.

Stderr on failure:
- `ERROR: <message>`
- `SECURITY ISSUE FOUND — INSTALLATION BLOCKED` followed by details

## Exit codes

- `0` — success
- `1` — error or security block
- `2` — cancelled
