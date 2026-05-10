# Filesystem Layout

Every path skilltap touches.

## Directory map

```
~/.config/skilltap/                       (global config dir — XDG_CONFIG_HOME/skilltap)
  config.toml                             global config
  state.json                              global state — skills, plugins, mcpServers
  taps/                                   cloned tap repos
    <tap-name>/
      tap.json
  cache/                                  cloned skill/plugin/mcp repos (by hash)
    <bunhash>/
  plugin-data/                            persistent plugin data dirs
    <plugin-name>/                        expanded as ${CLAUDE_PLUGIN_DATA}
  update-check.json                       self-update cache
  skills-update-check.json                skill-update cache

  config.toml.v1.bak                      legacy backups (after `migrate`)
  installed.json.v1.bak
  plugins.json.v1.bak

~/.agents/skills/                         (canonical global skill directories)
  <skill-name>/
    SKILL.md
    ...
  .disabled/
    <skill-name>/                         (disabled skills moved here)

<project>/                                (project root: directory containing .git or skilltap.toml)
  skilltap.toml                           project manifest (committed)
  skilltap.lock                           project lockfile (committed)
  .agents/
    state.json                            project state
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
<project>/.claude/skills/<name>   ->  <project>/.agents/skills/<name>
<project>/.cursor/skills/<name>   ->  <project>/.agents/skills/<name>
<project>/.codex/skills/<name>    ->  <project>/.agents/skills/<name>
<project>/.gemini/skills/<name>   ->  <project>/.agents/skills/<name>
<project>/.windsurf/skills/<name> ->  <project>/.agents/skills/<name>

# Claude Code agent definition files (COPIES, not symlinks)
~/.claude/agents/<name>.md
~/.claude/agents/.disabled/<name>.md      (when disabled)
<project>/.claude/agents/<name>.md
<project>/.claude/agents/.disabled/<name>.md

# MCP injection: per-agent config files modified by plugin and standalone-mcp install
~/.claude/settings.json                   (Claude Code global MCP config)
~/.cursor/mcp.json                        (Cursor)
~/.codex/mcp.json                         (Codex)
~/.gemini/settings.json                   (Gemini)
~/.windsurf/mcp.json                      (Windsurf)

# Project-scope MCP injection
<project>/.claude/settings.json
<project>/.cursor/mcp.json
<project>/.codex/mcp.json
<project>/.gemini/settings.json
<project>/.windsurf/mcp.json

# Backup files (one per modified config file, written before first modification)
~/.claude/settings.json.skilltap.bak
~/.cursor/mcp.json.skilltap.bak
~/.codex/mcp.json.skilltap.bak
~/.gemini/settings.json.skilltap.bak
~/.windsurf/mcp.json.skilltap.bak
```

## Project root resolution

```
findProjectRoot()  → walks up from process.cwd() looking for `.git`, falls back to cwd
findManifestRoot() → walks up looking for `skilltap.toml`, falls back to git root
```

`install`, `remove`, `update` use `findProjectRoot()`. `sync` uses `findManifestRoot()` and errors if neither exists.

This means project installs go to the git root, not necessarily the cwd.

## globalBase() and SKILLTAP_HOME

The base directory for global-scope operations is:

```
process.env.SKILLTAP_HOME ?? os.homedir()
```

Setting `SKILLTAP_HOME` redirects all global paths:
- `SKILLTAP_HOME=/tmp/test` → global skills at `/tmp/test/.agents/skills/`
- `SKILLTAP_HOME=/tmp/test` → global config at `/tmp/test/.config/skilltap/` (XDG still applies within this base)

Used in tests for isolation. `XDG_CONFIG_HOME` is independent — `SKILLTAP_HOME` controls home-relative paths; `XDG_CONFIG_HOME` controls the config dir specifically.

## Canonical skill location

A skill always lives at `.agents/skills/<name>/` — either global (`~/.agents/skills/<name>/`) or project (`<project>/.agents/skills/<name>/`). Per-agent directories (`.claude/skills/`, `.cursor/skills/`, etc.) hold symlinks pointing here.

When disabled, the skill is moved to `.agents/skills/.disabled/<name>/` and per-agent symlinks are removed. On re-enable, the directory is moved back and symlinks recreated.

## Linked skills

`scope: linked` skills are symlinks at `.agents/skills/<name>` pointing to an absolute external path stored in the record's `path` field. The external directory is owned by the user and is never deleted by skilltap.

## Plugin-owned content

Plugin-owned skills live at the same canonical path as standalones (`.agents/skills/<name>/`) but are recorded in `state.plugins[].components[]`, NOT in `state.skills[]`. Use `info <plugin-name>` to enumerate components or `state.json` directly.

## Agent definition files

Agent definitions (from plugin components of type `"agent"`) are **copies**, not symlinks — different platforms may have different formats and skilltap must avoid accidental shared mutation.

When disabled, the file is moved to `.claude/agents/.disabled/<name>.md`. When toggled back on, moved back.

## MCP injection namespace

MCP servers injected by skilltap use this format in the agent config:

- Plugin-owned: `skilltap:<plugin-name>:<server-name>`
- Standalone (from `install mcp <source>`): `skilltap:standalone:<install-name>`

Example `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "skilltap:dev-toolkit:database": { ... },
    "skilltap:standalone:search":    { ... },
    "my-own-mcp": { ... }
  }
}
```

On removal or toggle-off, only entries matching the appropriate `skilltap:` prefix are removed. User-added entries (`my-own-mcp`) are untouched.

## Backup files

Before the **first** modification to any agent config file (by any plugin / mcp install), a backup is written:

```
<config-path>.skilltap.bak
```

This is a one-time snapshot of the pre-skilltap state. Subsequent installs / removals do not overwrite the backup. To restore: `cp <config-path>.skilltap.bak <config-path>`.

## Cache directory

Repos cloned during install are cached at:

```
~/.config/skilltap/cache/<hash>/
```

The hash is `Bun.hash(repoUrl).toString(16)` — a hex string derived from the repo URL. Used to avoid re-cloning on subsequent operations and to compute update diffs.

## XDG_CONFIG_HOME

If `$XDG_CONFIG_HOME` is set, skilltap uses `$XDG_CONFIG_HOME/skilltap/` instead of `~/.config/skilltap/`. `SKILLTAP_HOME` and `XDG_CONFIG_HOME` are independent — `SKILLTAP_HOME` controls the base for skill dirs and home-relative paths; `XDG_CONFIG_HOME` controls the config dir location only.

## Built-in tap clone

The built-in `skilltap-skills` tap is cloned lazily on first use to `~/.config/skilltap/taps/skilltap-skills/`. Disable with `builtin_tap = false` in config to skip the clone entirely.
