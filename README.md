# skilltap-skills

Official tap for [skilltap](https://github.com/nklisch/skilltap) — the skill, plugin, and MCP-server manager for AI agents.

## Skills

| Skill | What it covers |
|-------|----------------|
| `skilltap` | Installing, updating, removing, listing, toggling, adopting, moving, and configuring skills + plugins + standalone MCP servers. References for `config.toml` (flat `[security]` + `[scanner]` blocks), `state.json`, `skilltap.toml` + `skilltap.lock` (project manifest + lockfile), the filesystem layout, non-interactive use (TTY / plain / JSON output modes), and troubleshooting. |
| `skilltap-find` | Discovery via taps and the skills.sh registry. References for every install source format (`install <type> <source>`), git-only tap management, and trust tiers. |
| `skilltap-author` | Creating new skills, plugins, and taps. References for `SKILL.md` frontmatter, the native `.skilltap/<name>.toml` plugin format (plus Claude Code + Codex), `tap.json` indexes, and publishing (git, npm with provenance, taps). |

This tap ships **built-in** with skilltap — no `tap add` is required. Every install of skilltap automatically searches it (lazily cloned to `~/.config/skilltap/taps/skilltap-skills/`).

## Install the skills

```bash
skilltap install skill skilltap        --scope global --yes
skilltap install skill skilltap-find   --scope global --yes
skilltap install skill skilltap-author --scope global --yes
```

Add `--also claude-code` (or `cursor`, `codex`, `gemini`, `windsurf`) on each command if you want symlinks into the per-agent skills directory.

## What is skilltap?

[skilltap](https://github.com/nklisch/skilltap) is a CLI for installing agent skills (SKILL.md files), plugins (skills + MCPs + agent definitions), and standalone MCP servers from any git host, npm, or local path — Homebrew taps for AI agents.

Three artifact types, addressed explicitly:

```bash
skilltap install skill   <source>     # single SKILL.md or multi-skill repo
skilltap install plugin  <source>     # bundle of skills + MCPs + agents
skilltap install mcp     <source>     # standalone MCP server
```

Skills are agent-agnostic and install to `.agents/skills/`, with optional symlinks into per-agent directories (`.claude/skills/`, `.cursor/skills/`, etc.) via `--also <agent>`. Plugins inject MCP configs into per-agent settings files (e.g. `~/.claude/settings.json`) namespaced as `skilltap:<plugin>:<server>` so user-added MCPs are never disturbed.

skilltap auto-detects scope from the cwd: inside a git repo → `project` (writes to `<project>/.agents/`); outside → `global` (writes to `~/.agents/`). Override with `--scope project|global`.

For team projects, skilltap maintains `skilltap.toml` + `skilltap.lock` at the project root — commit both, and teammates can run `skilltap sync --apply` to bring their machine to parity.

## Authoring

If you want to add your own skills to a tap or run a tap of your own, install the `skilltap-author` skill and ask your agent to walk you through it — or read the reference pages directly:

- [SKILL.md frontmatter](.agents/skills/skilltap-author/reference/skill-format.md)
- [Plugin manifests](.agents/skills/skilltap-author/reference/plugin-format.md) (native `.skilltap/<name>.toml`, Claude Code, Codex)
- [tap.json schema](.agents/skills/skilltap-author/reference/tap-format.md)
- [Publishing](.agents/skills/skilltap-author/reference/publishing.md)
