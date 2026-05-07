# skilltap-skills

Official tap for [skilltap](https://github.com/nklisch/skilltap) — the skill and plugin manager for AI agents.

## Skills

| Skill | Description |
|-------|-------------|
| `skilltap` | Install, update, remove, list, and manage skills and plugins (incl. MCP servers + agent definitions) |
| `skilltap-find` | Search taps and the skills.sh registry to discover new skills and plugins |

## Usage

This tap ships **built-in** with skilltap — no `tap add` is required.

```bash
# Install the skills directly from the built-in tap
skilltap install skilltap --global --yes
skilltap install skilltap-find --global --yes
```

## What is skilltap?

[skilltap](https://github.com/nklisch/skilltap) is a CLI for installing agent skills (SKILL.md files) and plugins from any git host, npm, or HTTP registry — Homebrew taps for AI agents. Skills are agent-agnostic and install to `.agents/skills/`. Plugins bundle skills together with MCP servers and agent definitions, and inject MCP configs into per-agent settings (`~/.claude/settings.json`, `~/.cursor/mcp.json`, etc.).
