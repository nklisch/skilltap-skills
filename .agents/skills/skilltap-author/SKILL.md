---
name: skilltap-author
description: >
  Create, scaffold, write, author, validate, publish, and share agent skills,
  plugins, and taps for the skilltap CLI. Use when the user wants to write a
  new skill, design a multi-skill repo, build a plugin manifest (native
  skilltap, Claude Code, or Codex), create or maintain a tap, validate
  content before pushing (`skilltap doctor skill <path>` / `doctor plugin
  <path>`), publish to npm with provenance, set up GitHub Artifact
  Attestations, or register their work in a tap index.
license: MIT
---

# skilltap-author

## What this skill enables

- **Scaffold** a skill, plugin, or tap with `skilltap create` or `skilltap tap init`.
- **Validate** before sharing with `skilltap doctor skill <path>` (or `doctor plugin <path>`).
- **Publish** via git host, npm (with provenance), or a tap.
- **Author** SKILL.md frontmatter, `.skilltap/<name>.toml` plugin manifests, and `tap.json` indexes correctly on the first try.

## Quickstart

```bash
# 1. Scaffold
skilltap create my-skill --template basic

# 2. Edit SKILL.md — frontmatter.name MUST match the directory name
cd my-skill && $EDITOR SKILL.md

# 3. Validate
skilltap doctor skill .

# 4. Publish
git init && git add . && git commit -m "Initial commit"
git remote add origin https://github.com/you/my-skill
git push -u origin main
git tag v1.0.0 && git push --tags
```

Users can then install with `skilltap install skill github:you/my-skill`.

## CLI reference

```
skilltap create [name] [--template basic|npm|multi] [--dir <path>]
  Scaffold a new skill. Prompts for name/description/license/author in TTY.
  When both --name (positional) and --template are provided, runs
  non-interactively with sensible defaults.

  Templates:
    basic — SKILL.md + .gitignore (single-skill repo)
    npm   — SKILL.md + package.json + .gitignore + .github/workflows/publish.yml
    multi — SKILL.md files under .agents/skills/<name>/ + .gitignore
            (for multi-skill repos with a reference/ subdirectory)

skilltap doctor skill <path>     # validate a single skill
skilltap doctor plugin <path>    # validate a plugin
skilltap doctor [--fix] [--json] # full env + state health check
```

`skilltap verify <path>` was removed in v2 — running it prints a hint pointing at `doctor skill <path>` (or `doctor plugin <path>` for plugins).

`skilltap tap init <directory>` scaffolds a new tap repo (skeleton `tap.json` + `.git/`).

## Decision map

| Asked to… | Go to |
|---|---|
| Write a SKILL.md from scratch | [reference/skill-format.md](reference/skill-format.md) |
| Build a plugin (bundle skills + MCPs + agents) | [reference/plugin-format.md](reference/plugin-format.md) |
| Maintain or create a tap (curated index) | [reference/tap-format.md](reference/tap-format.md) |
| Publish via npm with provenance, or set up GitHub Artifact Attestations | [reference/publishing.md](reference/publishing.md) |
| Install / update / remove / manage skills | `../skilltap/SKILL.md` |
| Find or search for skills | `../skilltap-find/SKILL.md` |

## Plugin formats overview

skilltap reads three plugin manifest formats. Pick one based on your audience.

| Format | Manifest path | When to use |
|---|---|---|
| **skilltap (native)** | `.skilltap/<name>.toml` | Recommended for new plugins. TOML, multi-plugin friendly, has explicit `publish = true` opt-in, supports stdio + http MCPs, Claude Code agent definitions. |
| **Claude Code** | `.claude-plugin/plugin.json` | Adopt an existing Claude Code plugin without rewriting its manifest. |
| **Codex** | `.codex-plugin/plugin.json` | Existing Codex plugin. |

Detection runs in that order at install time. All three normalize to the same internal `PluginManifest`. See [reference/plugin-format.md](reference/plugin-format.md) for full schemas.

## Pre-publish checklist

1. `skilltap doctor skill <path>` (or `doctor plugin <path>`) exits 0.
2. `frontmatter.name` matches the directory name exactly.
3. Description is action-oriented and exhaustive — test that your agent loads the skill on plain-language prompts.
4. Static security scan clean (no anti-trojan-source / dangerous-pattern warnings).
5. Total skill size < 50 KB (configurable via `[scanner].max_size`).
6. `README.md` for humans browsing the repo (separate from `SKILL.md`).
7. Release tag: `git tag v1.0.0 && git push --tags`.
8. For npm publishing: `npm publish --access public --provenance` from a public GitHub Actions workflow with `id-token: write`.
