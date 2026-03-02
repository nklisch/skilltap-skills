---
name: skilltap
description: Manage agent skills with the skilltap CLI — install, update, remove, and list skills from any git host or npm.
license: MIT
---
# skilltap

skilltap is a CLI for installing agent skills (SKILL.md files) from any git host or npm. Skills install to `.agents/skills/` (project) or `~/.agents/skills/` (global) and are agent-agnostic.

## Install

```bash
skilltap install <source> [flags]
```

### Source formats

```bash
skilltap install user/repo                        # GitHub shorthand
skilltap install github:user/repo                 # GitHub explicit
skilltap install https://gitea.example.com/u/r   # Any git URL
skilltap install git@github.com:user/repo.git     # SSH
skilltap install commit-helper                    # By tap name
skilltap install commit-helper@v1.2.0             # Tap name + version/ref
skilltap install ./my-skill                       # Local path
skilltap install npm:@scope/skill-name            # npm registry
skilltap install npm:@scope/skill-name@1.2.3      # npm pinned version
```

### Key flags

| Flag | Effect |
|------|--------|
| `--global` | Install to `~/.agents/skills/` |
| `--project` | Install to `.agents/skills/` in current project |
| `--yes` | Auto-select all skills; auto-accept clean scans |
| `--also <agent>` | Also symlink to agent dir (repeatable) |
| `--ref <ref>` | Branch or tag to install |
| `--strict` | Abort on any security warning (exit 1) |
| `--semantic` | Run Layer 2 LLM-based semantic scan |
| `--skip-scan` | Skip security scanning |

`--yes` does **not** skip the scope prompt — use `--yes --global` or `--yes --project` for fully non-interactive installs.

`--also` values: `claude-code`, `cursor`, `codex`, `gemini`, `windsurf`

### Scope behavior

- Without `--global` or `--project`: prompts "Install to: Global / Project"
- `--project` requires a git repo root to be found in cwd or parents

## List installed skills

```bash
skilltap list              # table: name, scope, source, trust, installed date
skilltap list --json       # machine-readable
```

## Update

```bash
skilltap update            # update all installed skills
skilltap update <name>     # update one skill
skilltap update --yes      # non-interactive (auto-accept clean updates)
```

## Remove

```bash
skilltap remove <name>
skilltap remove <name> --yes    # skip confirmation
```

## Info

```bash
skilltap info <name>    # source, path, version/SHA, trust tier, agent symlinks
```

## Link (local development)

Symlink a local skill directory instead of installing from a remote source. Changes to the local directory are reflected immediately.

```bash
skilltap link ./my-skill               # prompts for scope
skilltap link ./my-skill --global
skilltap link ./my-skill --project
skilltap unlink my-skill
```

## Doctor

Diagnose environment: git, config, dirs, installed.json integrity, symlinks, taps, agents, npm.

```bash
skilltap doctor
skilltap doctor --fix    # auto-repair where safe
```

## Config

Interactive setup wizard for `~/.config/skilltap/config.toml`.

```bash
skilltap config
```

## Exit codes

- `0` — success
- `1` — error
- `2` — user cancelled
