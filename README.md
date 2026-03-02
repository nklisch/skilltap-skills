# skilltap-skills

Official tap for [skilltap](https://github.com/nklisch/skilltap) — the skill manager for AI agents.

## Skills

| Skill | Description |
|-------|-------------|
| `skilltap` | Install, update, remove, and list agent skills |
| `skilltap-find` | Search taps and npm to discover new skills |

## Usage

```bash
# Add this tap
skilltap tap add skilltap https://github.com/nklisch/skilltap-skills

# Install the skills
skilltap install skilltap --global --yes
skilltap install skilltap-find --global --yes
```

## What is skilltap?

[skilltap](https://github.com/nklisch/skilltap) is a CLI for installing agent skills (SKILL.md files) from any git host or npm — Homebrew taps for AI agents. Skills are agent-agnostic and install to `.agents/skills/`.
