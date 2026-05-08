# skilltap-skills

Official tap for [skilltap](https://github.com/nklisch/skilltap) — the skill and plugin manager for AI agents.

## Skills

| Skill | What it covers |
|-------|----------------|
| `skilltap` | Installing, updating, removing, listing, and configuring skills + plugins. Includes references for `config.toml`, `installed.json`, `plugins.json`, the filesystem layout, agent-mode rules, and troubleshooting. |
| `skilltap-find` | Discovery via taps and the skills.sh registry. Includes references for every install source format, tap management (git + HTTP), and trust tiers. |
| `skilltap-author` | Creating new skills, plugins, and taps. Includes references for `SKILL.md` frontmatter, Claude Code + Codex plugin manifests, `tap.json` indexes, and publishing (git, npm with provenance, taps). |

This tap ships **built-in** with skilltap — no `tap add` is required. Every install of skilltap automatically searches it.

## Install the skills

```bash
skilltap install skilltap --global --yes
skilltap install skilltap-find --global --yes
skilltap install skilltap-author --global --yes
```

## What is skilltap?

[skilltap](https://github.com/nklisch/skilltap) is a CLI for installing agent skills (SKILL.md files) and plugins from any git host, npm, or HTTP registry — Homebrew taps for AI agents.

Skills are agent-agnostic and install to `.agents/skills/`, with optional symlinks into per-agent directories (`.claude/skills/`, `.cursor/skills/`, etc.). Plugins bundle skills together with MCP servers and agent definitions, and inject MCP configs into per-agent settings files.

## Authoring

If you want to add your own skills to a tap or run a tap of your own, install the `skilltap-author` skill and ask your agent to walk you through it — or read the reference pages directly:

- [SKILL.md frontmatter](.agents/skills/skilltap-author/reference/skill-format.md)
- [Plugin manifests](.agents/skills/skilltap-author/reference/plugin-format.md)
- [tap.json schema](.agents/skills/skilltap-author/reference/tap-format.md)
- [Publishing](.agents/skills/skilltap-author/reference/publishing.md)
