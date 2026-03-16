---
name: skilltap
description: Manage agent skills with the skilltap CLI — install, update, remove, disable, enable, adopt, and move skills from any git host or npm. Use when the user wants to install, update, remove, list, adopt, move, disable, or enable agent skills, or configure skilltap settings.
license: MIT
---
# skilltap

Agent-mode CLI for installing, updating, removing, and managing agent skills. All output is plain text, no interactive prompts, security warnings hard-fail.

## Before using any command

Run `skilltap status` first. If `agent-mode` is disabled, **stop** and tell the user:

> "skilltap agent mode is not enabled. Please run `skilltap config agent-mode` to configure it, then try again."

Do not attempt any skill operations until agent mode is confirmed enabled.

If the user wants to understand what skilltap is, how it works, or needs help with human-mode commands (interactive wizards, config setup), see [reference/human-guide.md](reference/human-guide.md).

## Agent-mode behavior

- `--yes` is auto-applied — never pass it
- `--strict` is forced — never pass it
- `--skip-scan` is rejected (exit 1) when `require_scan` is enabled
- Scope must be set via `--project` / `--global` or pre-configured in `agent-mode.scope`
- Security warnings always hard-fail (exit 1) — no override available
- All output is plain text to stdout; errors to stderr
- All skill names in multi-skill repos are installed automatically

## Command reference

| Task | Command |
|------|---------|
| Check status | `skilltap status [--json]` |
| Install | `skilltap install <source> [--project\|--global]` |
| Update | `skilltap update [name] [--json]` |
| Remove | `skilltap skills remove <name>` |
| List | `skilltap skills [--json]` |
| Info | `skilltap skills info <name>` |
| Disable | `skilltap skills disable <name>` |
| Enable | `skilltap skills enable <name>` |
| Adopt | `skilltap skills adopt [name...] [--yes]` |
| Move | `skilltap skills move <name> --global\|--project` |
| Find | `skilltap find [query] [--json]` |

## Status

```bash
skilltap status --json
```

Fields: `agentMode`, `scope`, `security.human`, `security.agent`, `agent_cli`, `also`, `taps`

Always check `agentMode` is `true` before proceeding.

## Install

```bash
skilltap install <source> --project
skilltap install <source> --global
skilltap install <source> --also claude-code --also cursor
skilltap install <source> --ref v1.2.0
skilltap install <source> --semantic
```

Source formats: `user/repo`, `github:user/repo`, any git URL, SSH URL, `skill-name` (tap lookup), `skill-name@v1.2.0`, `npm:@scope/pkg`, `npm:@scope/pkg@1.2.3`

## Update

```bash
skilltap update                       # all installed skills
skilltap update <name>                # single skill
skilltap update --semantic            # with semantic security scan
skilltap update --json                # JSON output
```

Warnings on update skip the affected skill and continue. Disabled skills are skipped.

## Skills command group

Unified skill management — list, adopt, move, disable, enable, remove, info. See [reference/skills-commands.md](reference/skills-commands.md) for full command details and flags.

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

Stdout, one line per skill:
- `OK: Installed <name> -> <path>`
- `OK: <name> is already up to date.`
- `SKIP: <name> is linked.`
- `SKIP: <name> — disabled`

Stderr on failure:
- `ERROR: <message>`
- `SECURITY ISSUE FOUND — INSTALLATION BLOCKED` followed by details

## Exit codes

- `0` — success
- `1` — error or security block
- `2` — cancelled
