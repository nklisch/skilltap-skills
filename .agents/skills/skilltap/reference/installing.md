# Installing Skills, Plugins, and MCP Servers

Every install is a typed subcommand:

```bash
skilltap install skill   <source> [flags]
skilltap install plugin  <source> [flags]
skilltap install mcp     <source> [flags]
```

The type is required — there is no auto-detect across types. If a source has a SKILL.md but no plugin manifest, `install plugin` errors with a hint to use `install skill` (and vice versa).

## Source formats

The same source forms work for `install skill`, `install plugin`, and `install mcp`:

| Form | Example | Notes |
|---|---|---|
| Tap-resolved name | `commit-helper` | Bare name; searches all configured taps. Fails fast if no taps. |
| Tap name + version | `commit-helper@v1.2.0` | Pins to a git ref. |
| Tap-defined plugin | `tap-name/plugin-name` | First segment must match a known tap name exactly. |
| GitHub shorthand | `owner/repo` | Resolves via `default_git_host` (default `https://github.com`). |
| GitHub explicit | `github:owner/repo` | Always GitHub regardless of `default_git_host`. |
| GitHub @ ref | `owner/repo@v1.2.0` | Branch or tag suffix. |
| Plugin selector | `owner/repo:plugin-name` | Plugin only — picks one plugin from a multi-plugin repo. |
| Plugin all | `owner/repo:*` | Plugin only — installs every publishable plugin in the repo. |
| Plugin @ ref + selector | `owner/repo@v1.0:frontend` | Both can compose. |
| Any HTTPS URL | `https://gitea.example.com/x/y` | Any git host. |
| SSH URL | `git@github.com:owner/repo.git` | Uses your local git auth. |
| npm package | `npm:@scope/name` | npm registry. Pin with `@version` suffix: `npm:@scope/name@1.2.3` or `npm:@scope/name@^1.0`. |
| Local directory | `./path` or `/abs` or `~/path` | Symlinked, not copied. |

## Scope: smart default

When `--scope` is not passed and `[defaults] scope` is empty in config, scope is inferred from the cwd:

- **Inside a git repo** → `project` → installs to `<project>/.agents/skills/{name}/`.
- **Outside a git repo** → `global` → installs to `~/.agents/skills/{name}/`.

The inferred scope is reported in install output (e.g. `scope: project (inferred from cwd)`). Override with `--scope project` or `--scope global`.

`install skill`, `install plugin`, and `install mcp` all honor smart-scope.

## Per-agent symlinks: `--also`

Use `--also <agent>` to create a symlink in the per-agent skills directory so that agent loads the skill. Repeatable for multiple agents.

Valid agent IDs (single source of truth in `core/src/symlink.ts`):

| ID | Global symlink | Project symlink |
|---|---|---|
| `claude-code` | `~/.claude/skills/<name>` | `<project>/.claude/skills/<name>` |
| `cursor` | `~/.cursor/skills/<name>` | `<project>/.cursor/skills/<name>` |
| `codex` | `~/.codex/skills/<name>` | `<project>/.codex/skills/<name>` |
| `gemini` | `~/.gemini/skills/<name>` | `<project>/.gemini/skills/<name>` |
| `windsurf` | `~/.windsurf/skills/<name>` | `<project>/.windsurf/skills/<name>` |

If `[defaults] also = ["claude-code"]` is set in config, every install applies that list automatically.

For MCP installs, `--also` injects the MCP entry into each agent's settings file (`~/.claude/settings.json`, `~/.cursor/mcp.json`, etc.) instead of creating a symlink.

## Install flags

| Flag | skill | plugin | mcp | Effect |
|---|---|---|---|---|
| `--scope project\|global` | ✓ | ✓ | ✓ | Override smart-scope inference. |
| `--also <agent>` (repeatable) | ✓ | ✓ | ✓ | Per-agent symlink (skills) or config injection (mcp). |
| `--ref <ref>` | ✓ | ✓ | — | Branch / tag for git sources. |
| `--yes`, `-y` | ✓ | ✓ | ✓ | Auto-accept prompts (multi-skill selection, "install?", "save defaults?"). Does NOT bypass scan warnings. |
| `--strict` | ✓ | ✓ | — | Hard-fail on any security warning (one-shot `on_warn = "fail"`). |
| `--skip-scan` | ✓ | ✓ | — | Skip security scanning entirely. Use only for sources you've vetted. |
| `--semantic` | ✓ | ✓ | — | Force Layer 2 semantic scan in addition to static. |
| `--quiet` | ✓ | ✓ | ✓ | Suppress per-step install details (overrides `verbose = true`). |
| `--json` | ✓ | ✓ | ✓ | Newline-delimited JSON event output. Auto-selected when stdout is not a TTY. |
| `--force-capture` | — | ✓ | — | Auto-capture every same-source standalone match into the new plugin (non-interactive). |
| `--no-capture` | — | ✓ | — | Disable plugin capture; standalones stay in place. Mutually exclusive with `--force-capture`. |

## Decision tree at install time

```
source
  │
  ├── scope? ┬── --scope project ─→ project
  │          ├── --scope global ──→ global
  │          ├── [defaults].scope set ─→ that value
  │          └── neither ─────────→ smart default: in git repo → project, else global
  │
  ├── agents? ┬── --also passed ────────────────→ use flag value(s)
  │           ├── --yes ──────────────────────→ use [defaults].also
  │           ├── [defaults].also set ────────→ use it (no prompt)
  │           └── none of the above + TTY ────→ prompt "Which agents?"
  │
  → resolve adapter (tap → git URL → npm → local → github shorthand)
  → clone / fetch / copy
  → discover skills (skill) or detect plugin manifest (plugin)
                 │
                 ├── single skill ────→ auto-select
                 ├── multi + --yes ───→ auto-select all
                 └── multi + TTY ─────→ prompt "Which skills to install?"
                                                     │
                                                ┌─ source matches [security].trust glob? → skip scan ─┐
                                                │ OR --skip-scan? OR [security].scan = "none"?         │
                                                │                                                       │
                                                → static scan (Layer 1)                                 │
                                                │                                                       │
                                                ├─ clean ────────────────────────────────────────────►─┤
                                                │                                                       │
                                                ├─ warnings ┬── --strict OR on_warn = "fail" → ABORT (exit 1)
                                                │           ├── on_warn = "install" → log + proceed
                                                │           └── on_warn = "prompt" → "Install anyway?" (TTY)
                                                │                                                       │
                                                └─ --semantic OR scan = "semantic" → Layer 2            │
                                                                                                        │
                                                     ▼
                                                ── --yes? ──→ install silently
                                                └── else + TTY → prompt "Install? (Y/n)"
```

## Multi-skill repos

A repo can contain multiple `SKILL.md` files. Discovery scans these locations in priority order: `SKILL.md` at root → `.agents/skills/*/SKILL.md` → `skills/*/SKILL.md` → `plugins/*/skills/*/SKILL.md` → `.{claude,cursor,codex,gemini,windsurf}/skills/*/SKILL.md` → deep `**/SKILL.md` (deep scan triggers a confirmation prompt unless `--yes`).

In TTY mode, the user picks which skills to install. With `--yes` (or non-TTY mode), all discovered skills are auto-selected.

## Plugin auto-detection

For `install plugin`, detection runs **before** skill scanning, in this priority order:

1. `.skilltap/<name>.toml` — native skilltap plugin format. Multi-plugin repos use multiple `.toml` files here.
2. `.claude-plugin/plugin.json` — Claude Code plugin format.
3. `.codex-plugin/plugin.json` — Codex plugin format.

If none found, `install plugin` errors with a hint to use `install skill`.

Plugin components installed:

- **Skills** → `.agents/skills/<name>/`, recorded under `state.plugins[].components[]` (NOT in `state.skills[]`).
- **MCP servers** → injected into per-agent config files, namespaced as `skilltap:<plugin>:<server>`. Two server types: `stdio` (command/args/env) and `http` (url/headers).
- **Agent definitions** → `.claude/agents/<name>.md` (Claude Code only currently). These are file copies, not symlinks.

A backup of each agent config file is written before the **first** modification: `<config-path>.skilltap.bak` (e.g. `~/.claude/settings.json.skilltap.bak`). One-time backup; subsequent plugin changes don't overwrite it.

Removal preserves entries not namespaced with `skilltap:` (user-added MCP servers are never touched).

## Multi-plugin repos

A repo can publish more than one plugin via multiple `.skilltap/<name>.toml` manifests with `publish = true`:

| Form | Behavior |
|---|---|
| `install plugin owner/repo` | Single-plugin repo: installs that one. Multi-plugin repo: TTY picker, or non-TTY error `multiple plugins available: <name1>, <name2>; specify with owner/repo:<name>`. |
| `install plugin owner/repo:plugin-name` | Installs the named plugin directly. |
| `install plugin owner/repo:*` | Installs every publishable plugin from the repo. |

Selectors compose with `@ref` and full URLs: `owner/repo@v1.2.0:frontend`, `https://gitea.example.com/owner/repo:*`, `git@github.com:owner/repo.git:frontend`. The `:plugin-name` parser splits off the **last** `:` after stripping `@ref`, so HTTPS URLs (`https://...`) are unaffected.

## Plugin capture

When installing a plugin whose components match standalones already installed from the same canonical source, skilltap offers to **capture** them into the new plugin record:

- **TTY**: prompts `Capture these into the plugin? (Y/n)`.
- **`--yes`**: auto-confirms same-source matches.
- **`--force-capture`**: auto-captures, including cross-source matches (different source URL but same name).
- **`--no-capture`**: skip capture entirely; install side-by-side.

Capture is atomic: removes the standalone records, prunes the standalone's namespaced MCP keys, removes old symlinks, then writes the plugin record. If any step fails, rolls back to pre-capture state.

`--force-capture` and `--no-capture` are mutually exclusive — passing both errors.

## Output formats

### Plain mode (TTY off, `--json` not set)

```
OK: Installed <name> -> <path>
OK: Installed <name> -> <path> (<ref>)
OK: <name> is already up to date.
SKIP: <name> is linked.
SKIP: <name> — disabled
```

```
# Plugin install summary
OK: Installed plugin <name> (3 skills, 2 MCPs, 1 agent)
```

Errors on stderr:
```
ERROR: <message>
```

### TTY mode

Clack-styled steps, spinners, and confirm prompts. Same outcomes as plain mode.

### JSON mode (`--json`)

Newline-delimited JSON events, one per line. Each event has at minimum `{ type, ... }`. Suitable for piping into a parser. Always combined with non-zero exit on error.

## Edge cases

**Already installed at the same ref:** prints `OK: <name> is already up to date.`, exits 0.

**Existing directory conflict:** if a directory at the target path exists and is not managed by skilltap, install fails with `ERROR:`. Run `skilltap adopt <path>` to bring it under management first.

**Repo gone / network error:** exits 1 with `ERROR: <message>`. No partial state is written.

**Local path install:** `./path` creates a symlink to the absolute target — no copy. Record has `scope: linked` and `path` points to the target. Updates are skipped: `SKIP: <name> is linked.`

**npm install:** runs `npm pack` + extract into the cache, then copies to the install dir. Pin via `@version` suffix on the package name (`npm:@scope/name@1.2.3`). Updates compare version strings, not commit SHAs.

**Corrupt `skilltap.toml` in `--scope project`:** in TTY mode, install backs up to `skilltap.toml.bak` and resets to empty. In non-TTY mode, install refuses with a hint: `Run 'skilltap doctor --fix' to back up the corrupt manifest and reset to empty, then retry.`

**Source matches `[security].trust` glob:** scan is skipped entirely (equivalent to `--skip-scan` for that one source). Use only for sources you've vetted at the org level.

**Security warnings under `--yes`:** `--yes` does NOT auto-confirm scan warnings. Behavior is governed by `[security].on_warn`:

| Setting | Behavior on warnings (with `--yes`) |
|---|---|
| `prompt` (default in TTY) | Non-TTY → fail. TTY → prompt despite `--yes`. |
| `install` | Log warnings, proceed. |
| `fail` (or `--strict`) | Hard fail, exit 1. |

The full warning block (file:line, raw + visible content, decoded base64, etc.) is printed to stderr before the prompt or failure. See [non-interactive.md](non-interactive.md) for handling under automation.
