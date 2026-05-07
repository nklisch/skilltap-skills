# skilltap — Human guide

Use this reference when the user wants to understand skilltap, set it up for the first time, or use interactive/human-mode features. Walk them through the relevant sections.

## What is skilltap?

skilltap is a CLI tool for managing agent skills — reusable instruction bundles that teach AI agents (Claude Code, Cursor, Codex, Gemini, Windsurf) how to perform specific tasks. It handles installing from git repos, npm, or curated registries (taps), with built-in security scanning. Plugins bundle skills, MCP servers, and agent definitions into a single installable unit.

## First-time setup

Guide the user through these steps in order:

1. **Install skilltap** (if not installed):
   ```bash
   npm install -g skilltap
   ```

2. **Run the setup wizard**:
   ```bash
   skilltap config
   ```
   This interactively configures: default scope (global vs project), agent symlink targets, and security settings. For non-interactive editing use `skilltap config get` / `skilltap config set`.

3. **Enable agent mode** (so agents can use skilltap non-interactively):
   ```bash
   skilltap config agent-mode
   ```
   This sets up: agent-mode enabled flag, default scope for agent operations, and agent-specific security policy.

4. **Browse and install skills** — the official skilltap tap is built in, so no tap setup is required. Use `skilltap find` immediately to search it:
   ```bash
   skilltap find
   skilltap find --interactive   # type-ahead search with install picker
   ```
   To add other community taps:
   ```bash
   skilltap tap add <name> <url-or-registry>
   ```
   Taps can be git repos or HTTP registries — `skilltap tap add` auto-detects. To bulk-install (or remove) skills from configured taps via a multiselect picker:
   ```bash
   skilltap tap install
   ```

## Security configuration

Security has separate settings for human and agent modes. Humans configure via interactive wizard:

```bash
skilltap config security
```

This walks through:
- Which mode to configure (human, agent, or both)
- Preset selection (none, relaxed, standard, strict)
- Custom options if no preset fits (scan level, warning behavior, require-scan)
- Trust overrides for specific taps or source types

### Presets explained

| Preset | What it does |
|--------|-------------|
| **none** | No scanning at all — trust everything |
| **relaxed** | Static scan runs but warnings are ignored |
| **standard** | Static scan, prompts on warnings (recommended for humans) |
| **strict** | Semantic AI scan, blocks on any concern (recommended for agents) |

### Trust overrides

Allow relaxing or tightening security for specific sources:
- Trust a known tap: `skilltap config security --trust tap:home=relaxed`
- Be strict with npm: `skilltap config security --trust source:npm=strict`

## Interactive commands

These commands use interactive UI (prompts, spinners, colored output) and are only available in human mode:

- `skilltap config` — full setup wizard
- `skilltap config agent-mode` — agent mode toggle (TTY only; the only way to enable or disable agent mode)
- `skilltap config security` — security wizard
- `skilltap config telemetry status|enable|disable` — toggle anonymous usage data (OS, arch, command success/fail — no skill names or paths)
- `skilltap config get [key] [--json]` — non-interactive; read one or all config values; safe for scripts
- `skilltap config set <key> <value>` — non-interactive; write a config value; silent on success; safe for scripts
- `skilltap find --interactive` / `skilltap find -i` — type-ahead skill search with install picker
- `skilltap skills adopt` (without names) — multiselect picker for unmanaged skills
- `skilltap plugin toggle <name>` — interactive component picker for an installed plugin

## Understanding skill scopes

- **Global**: installed in `~/.agents/skills/`, available everywhere
- **Project**: installed in `./.agents/skills/`, available in the current repo only
- **Linked**: symlinked from a local directory (for development)

## Agent symlinks (`--also`)

When installing a skill, `--also <agent>` creates symlinks in agent-specific directories so multiple agents can discover the skill:

```bash
skilltap install user/repo --also claude-code --also cursor
```

Supported agents: `claude-code`, `cursor`, `codex`, `gemini`, `windsurf`

## Disabled vs removed skills

- **Disable**: hides the skill from agents temporarily — files move to `.agents/skills/.disabled/` but remain installed. Re-enable anytime.
- **Remove**: permanently deletes the skill and all symlinks.

## Plugins

A plugin is a bundle of skills, MCP servers, and (Claude Code) agent definitions installed as a single unit. skilltap supports both Claude Code plugins (`.claude-plugin/plugin.json`) and Codex plugins (`.codex-plugin/plugin.json`).

### Installing a plugin

Use the same `skilltap install` command — plugin detection is automatic:

```bash
skilltap install https://github.com/corp/dev-toolkit --also claude-code --also cursor
```

If a plugin manifest is found, skilltap prompts: `Install as plugin? (Y/n)`. Declining falls back to skill-only install. With `--yes`, the plugin is accepted automatically.

### MCP servers

MCP servers declared in the plugin are injected into each target agent's config file:

| Agent | Config file |
|-------|------------|
| Claude Code | `~/.claude/settings.json` |
| Cursor | `~/.cursor/mcp.json` |
| Codex | `~/.codex/mcp.json` |
| Gemini | `~/.gemini/settings.json` |
| Windsurf | `~/.windsurf/mcp.json` |

Both stdio and HTTP MCP server types are supported (HTTP support added in v0.10.8). Injected entries are namespaced as `skilltap:<plugin>:<server>` to avoid collisions with user-configured servers. Before any agent config file is first modified, a `.skilltap.bak` backup is written.

### Managing plugins

```bash
skilltap plugin                       # list installed plugins
skilltap plugin info <name>           # details and component status
skilltap plugin toggle <name>         # interactive component picker (enable/disable individual skills, MCPs, or agents)
skilltap plugin toggle <name> --mcps  # disable/enable all MCP servers in the plugin
skilltap plugin toggle <name> --skills
skilltap plugin toggle <name> --agents
skilltap plugin remove <name>         # remove plugin and all its components
```

Removing a plugin removes only the `skilltap:*`-namespaced MCP entries it owns. User-configured MCP entries are never touched.

## Adopting unmanaged skills

If skills were manually placed in agent directories (not installed via skilltap), the adopt command brings them under management:

```bash
skilltap skills adopt              # interactive picker
skilltap skills adopt my-skill     # by name
skilltap skills adopt my-skill --track-in-place  # don't move, just track
```

Flags: `--global`, `--project`, `--track-in-place`, `--also <agent>`, `--skip-scan`, `--yes`

## Other useful commands

- `skilltap doctor [--fix]` — check environment health (git, config, dirs, integrity, symlinks, taps). `--fix` auto-repairs issues where safe.
- `skilltap verify [path]` — validate a skill before sharing (frontmatter, security scan, size). Useful as a pre-push hook or CI step.
- `skilltap create [name]` — scaffold a new skill from a template (`--template basic|npm|multi`).
- `skilltap completions <shell>` — generate shell completions for bash, zsh, or fish. `--install` writes to the standard location.
- `skilltap self-update` — update the CLI binary; auto-detects from GitHub releases.
- `skilltap status [--json]` — show agent mode status and current settings (scope, scan level, taps, plugins).
