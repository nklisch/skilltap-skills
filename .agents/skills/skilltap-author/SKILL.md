---
name: skilltap-author
description: >
  Create, scaffold, write, author, validate, verify, publish, and share agent
  skills, plugins, and taps for the skilltap CLI. Use when the user wants to
  write a new skill, create a tap, understand what goes in SKILL.md, validate
  content before pushing, publish a skill to npm, set up provenance, build a
  plugin manifest, or register their work in a tap index.
license: MIT
---

# skilltap-author

## What this skill enables

- **Scaffold** a skill, plugin, or tap with `skilltap create` or `skilltap tap init`.
- **Validate** before sharing with `skilltap verify`.
- **Publish** via git host, npm (with provenance), or a tap.
- **Author** SKILL.md frontmatter, `plugin.json` manifests, and `tap.json` indexes correctly on the first try.

## Quickstart

```bash
# 1. Scaffold
skilltap create my-skill --template basic

# 2. Edit SKILL.md — name must match directory name
cd my-skill && $EDITOR SKILL.md

# 3. Validate
skilltap verify .

# 4. Publish
git init && git add . && git commit -m "Initial commit"
git remote add origin https://github.com/you/my-skill
git push -u origin main
git tag v1.0.0 && git push --tags
```

Users can then install with `skilltap install github:you/my-skill`.

## CLI reference

```
skilltap create [name] [--template basic|npm|multi]
  Scaffold a new skill. Prompts for name/description/license/author in TTY.
  With --template, runs non-interactive with sensible defaults.

  Templates:
    basic  — SKILL.md + .gitignore
    npm    — SKILL.md + package.json + .gitignore + .github/workflows/publish.yml
    multi  — SKILL.md files under .agents/skills/<name>/ + .gitignore
              (for multi-skill repos with a reference/ subdirectory)

skilltap verify [path] [--json] [--all]
  Validate a skill at path (defaults to current directory).
  Exit 0 = no errors. Exit 1 = errors found.
  --json  Output structured JSON (for CI).
  --all   Validate all installed skills.
```

## Decision map

Asked to write a skill? → [reference/skill-format.md](reference/skill-format.md)

Asked to build a plugin (bundling MCP servers, agents, or multiple skills)? → [reference/plugin-format.md](reference/plugin-format.md)

Asked to create or maintain a tap (curated index)? → [reference/tap-format.md](reference/tap-format.md)

Asked how to publish, share, or set up provenance/attestations? → [reference/publishing.md](reference/publishing.md)

Asked to install, update, or remove a skill? → [../skilltap/](../skilltap/)

Asked to find or search for skills? → [../skilltap-find/](../skilltap-find/)

## Pre-publish checklist

1. `skilltap verify` exits 0.
2. `frontmatter.name` matches the directory name exactly.
3. Description is action-oriented and exhaustive — test that your agent loads the skill on plain-language prompts.
4. Static scan clean (no anti-trojan-source or dangerous-pattern warnings).
5. Total skill size < 50 KB.
6. `README.md` for humans browsing the repo (separate from `SKILL.md`).
7. Release tag: `git tag v1.0.0 && git push --tags`.
