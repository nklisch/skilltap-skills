# Filesystem Layout

## Directory map

```
~/.config/skilltap/                       (global config dir — XDG_CONFIG_HOME/skilltap)
  config.toml                             global config
  installed.json                          global skill records
  plugins.json                            global plugin records
  taps/                                   cloned tap repos
    <tap-name>/
      tap.json
  cache/                                  cloned skill/plugin repos (by hash)
    <bunhash>/

~/.agents/skills/                         (canonical global skill directories)
  <skill-name>/
    SKILL.md
    ...
  .disabled/
    <skill-name>/                         (disabled skills moved here)

<project>/.agents/                        (project scope)
  installed.json
  plugins.json
  skills/
    <skill-name>/
    .disabled/
      <skill-name>/

# Per-agent symlink directories (all point into canonical .agents/skills/)
~/.claude/skills/<name>          ->  ~/.agents/skills/<name>
~/.cursor/skills/<name>          ->  ~/.agents/skills/<name>
~/.codex/skills/<name>           ->  ~/.agents/skills/<name>
~/.gemini/skills/<name>          ->  ~/.agents/skills/<name>
~/.windsurf/skills/<name>        ->  ~/.agents/skills/<name>

# Project-scope per-agent symlinks
<project>/.claude/skills/<name>  ->  <project>/.agents/skills/<name>
<project>/.cursor/skills/<name>  ->  <project>/.agents/skills/<name>
<project>/.codex/skills/<name>   ->  <project>/.agents/skills/<name>
<project>/.gemini/skills/<name>  ->  <project>/.agents/skills/<name>
<project>/.windsurf/skills/<name> -> <project>/.agents/skills/<name>

# Claude Code agent definition files (COPIES, not symlinks)
~/.claude/agents/<name>.md
~/.claude/agents/.disabled/<name>.md      (when disabled)
<project>/.claude/agents/<name>.md
<project>/.claude/agents/.disabled/<name>.md

# MCP injection: per-agent config files modified by plugin install
~/.claude/settings.json                   (Claude Code global MCP config)
~/.cursor/mcp.json                        (Cursor)
~/.codex/mcp.json                         (Codex)
~/.gemini/settings.json                   (Gemini)
~/.windsurf/mcp.json                      (Windsurf)

# Backup files (one per modified config file, written before first modification)
~/.claude/settings.json.skilltap.bak
~/.cursor/mcp.json.skilltap.bak
~/.codex/mcp.json.skilltap.bak
~/.gemini/settings.json.skilltap.bak
~/.windsurf/mcp.json.skilltap.bak
```

## globalBase() resolution

The base directory for global-scope operations is resolved as:

```
process.env.SKILLTAP_HOME ?? os.homedir()
```

Setting `SKILLTAP_HOME` redirects all global paths:
- `SKILLTAP_HOME=/tmp/test` → global skills at `/tmp/test/.agents/skills/`
- `SKILLTAP_HOME=/tmp/test` → global config at `/tmp/test/.config/skilltap/` (XDG still applies within this base)

This env var is primarily used in tests for isolation.

## findProjectRoot() semantics

Walks up from `process.cwd()` looking for a `.git` directory. Returns the first directory containing `.git`. If no `.git` is found after reaching the filesystem root, falls back to `process.cwd()`.

This means `--project` installs are placed relative to the git repository root, not necessarily the current working directory.

## Canonical skill location

The canonical location for a skill is always `.agents/skills/<name>/` — either global (`~/.agents/skills/<name>/`) or project (`<project>/.agents/skills/<name>/`). Agent-specific directories (`.claude/skills/`, `.cursor/skills/`, etc.) contain symlinks that point to this canonical location.

When a skill is disabled, it is moved to `.agents/skills/.disabled/<name>/`. All agent-dir symlinks are removed. On re-enable, the directory is moved back and symlinks are recreated.

## Agent definition files

Agent definitions (from plugin components of type `"agent"`) are **copies**, not symlinks. This is because different platforms may have different formats and to prevent accidental shared mutation.

When disabled, the file is moved to `.claude/agents/.disabled/<name>.md`. When the plugin component is toggled back on, it is moved back to `.claude/agents/<name>.md`.

## MCP injection namespace

MCP servers injected by skilltap plugins are keyed as `skilltap:<plugin-name>:<server-name>` in the agent config file. Example in `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "skilltap:dev-toolkit:database": { ... },
    "my-own-mcp": { ... }
  }
}
```

On plugin removal, only entries matching the `skilltap:<plugin-name>:` prefix are removed. User-added entries like `my-own-mcp` are untouched.

## Backup files

Before the **first** modification to any agent config file (by any plugin install), a backup is written:

```
<config-path>.skilltap.bak
```

This is a one-time snapshot of the pre-skilltap state. Subsequent plugin installs or removals do not overwrite the backup. To restore: copy the `.skilltap.bak` file back over the original.

## Cache directory

Repos cloned during install are cached at:

```
~/.config/skilltap/cache/<hash>/
```

The hash is `Bun.hash(repoUrl).toString(16)` — a hex string derived from the repo URL. The cache is used to avoid re-cloning on subsequent operations and for update diffs.

## Config dir XDG

If `$XDG_CONFIG_HOME` is set, skilltap uses `$XDG_CONFIG_HOME/skilltap/` instead of `~/.config/skilltap/`. `SKILLTAP_HOME` and `XDG_CONFIG_HOME` are independent — `SKILLTAP_HOME` controls the base for skill dirs and home paths, while `XDG_CONFIG_HOME` controls the config file location.
