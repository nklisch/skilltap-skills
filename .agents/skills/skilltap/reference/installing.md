# Installing Skills and Plugins

## Source formats

| Format | Example | Notes |
|---|---|---|
| GitHub shorthand | `user/repo` | Uses `default_git_host` config (default `https://github.com`) |
| GitHub explicit | `github:user/repo` | Always GitHub |
| Any HTTPS git URL | `https://gitea.example.com/x/y` | Any git host |
| SSH URL | `git@github.com:user/repo.git` | Uses SSH auth |
| Tap skill name | `skill-name` | Searches all configured taps |
| Tap skill, pinned | `skill-name@v1.2.0` | Pins to a ref |
| Tap plugin | `tap-name/plugin-name` | First segment matches a tap name |
| Local directory | `./local-path` | Links by default (no copy) |
| npm | `npm:@scope/skill-name` | From npm registry |
| npm pinned | `npm:@scope/skill-name@1.2.3` | Pinned npm version |

## Decision tree: choosing what to pass

1. **Does the user give a GitHub URL or `user/repo`?** → use shorthand or full URL.
2. **Does the user give a skill name?** → use bare name (tap lookup). If a tap name is specified as a prefix, use `tap-name/skill-name`.
3. **Is it a local directory?** → use `./path`. This creates a symlink, not a copy.
4. **Is it on npm?** → use `npm:@scope/name`.
5. **Does it need a specific branch or tag?** → add `--ref <ref>` (git sources only; use `@version` suffix for tap lookups).

## Choosing scope

Pass exactly one of:
- `--project` — installs to `.agents/skills/` in the current project
- `--global` — installs to `~/.agents/skills/`

If neither is passed, the configured default (`agent-mode.scope`, default `project`) is used. When in doubt, use `--project` for project-specific tools and `--global` for tools needed across all projects.

## Symlinking to agent directories (`--also`)

Use `--also <agent>` to create a symlink in the per-agent skills directory so that agent can load the skill. Repeatable for multiple agents.

Valid agent values: `claude-code`, `cursor`, `codex`, `gemini`, `windsurf`.

Example: `skilltap install user/repo --project --also claude-code --also cursor`

If the user asks to install for a specific agent (e.g. "install for Claude Code"), pass `--also claude-code`.

## Install flags reference

| Flag | Effect |
|---|---|
| `--project` | Install to project scope |
| `--global` | Install to global scope |
| `--also <agent>` | Symlink into per-agent dir (repeatable) |
| `--ref <ref>` | Branch or tag to checkout (git sources) |
| `--quiet` | Suppress per-step install details |
| `--json` | JSON output |
| `--yes` | Auto-accept (no-op in agent mode — already auto-applied) |
| `--strict` | Hard-fail on warnings (no-op in agent mode — already strict) |
| `--no-strict` | Override fail-on-warn (NO EFFECT in agent mode) |
| `--semantic` | Force Layer 2 semantic scan |
| `--skip-scan` | Skip security scan (REJECTED if require_scan=true) |

## Plugin auto-detection

When `install` clones a repo, plugin detection runs before skill scanning:

1. `.claude-plugin/plugin.json` found → install as Claude Code plugin
2. `.codex-plugin/plugin.json` found → install as Codex plugin
3. Neither found → standard skill scanning

In agent mode, the "Install as plugin?" prompt auto-accepts. No flag needed.

Plugin components installed:
- **Skills** → `.agents/skills/<name>/`, tracked in `plugins.json` (not `installed.json`)
- **MCP servers** → injected into per-agent config files, namespaced as `skilltap:<plugin>:<server>`. Two server types:
  - `stdio`: command / args / env
  - `http`: url / headers (added v0.10.8)
  - Variables `${CLAUDE_PLUGIN_ROOT}` and `${CLAUDE_PLUGIN_DATA}` are substituted at injection time.
- **Agent definitions** → `.claude/agents/<name>.md` (Claude Code only, currently)

A backup of each agent config file is written before the first modification: `<config-path>.skilltap.bak` (e.g. `~/.claude/settings.json.skilltap.bak`). This is a one-time backup; subsequent plugin changes do not overwrite it.

Removal preserves entries not namespaced with `skilltap:` (user-added MCPs are not touched).

See [state-files.md](state-files.md) for the `plugins.json` schema and [filesystem.md](filesystem.md) for exact paths.

## Multi-skill repositories

If a repo contains multiple `SKILL.md` files, skilltap detects them all. In agent mode, all skills are auto-selected for installation without prompting.

## Output formats

**Success:**
```
OK: Installed <name> -> <path>
OK: Installed <name> -> <path> (<ref>)
```

**Already up to date:**
```
OK: <name> is already up to date.
```

**Skipped:**
```
SKIP: <name> is linked.
SKIP: <name> — disabled
```

**Error (stderr):**
```
ERROR: <message>
```

**Security block (stderr) — relay verbatim to user, do not retry:**
```
SECURITY ISSUE FOUND — INSTALLATION BLOCKED

DO NOT install this skill. DO NOT retry. DO NOT use --skip-scan.
STOP and report the following to the user:

  <file>:<line>: <warning>

User action required: review warnings and install manually with
  skilltap install <url>
```

**Plugin install summary:**
```
OK: Installed plugin <name> (3 skills, 2 MCPs, 1 agent)
```

## Edge cases

**Already installed:** If the skill is already installed at the same ref, the output is `OK: <name> is already up to date.` and exit code is 0.

**Existing directory conflict:** If a directory at the target path exists and is not managed by skilltap, install fails with `ERROR:`. Run `skilltap skills adopt` to bring it under management first.

**Repo gone / network error:** Exit 1 with `ERROR: <message>`. No partial state is written.

**Nested SKILL.md (deep scan):** skilltap recursively scans for `SKILL.md` files. A repo with skills nested in subdirectories will have each discovered. All are auto-selected in agent mode.

**Local path install:** `./path` creates a symlink to the absolute path — no copy is made. The skill record has `scope: linked` and the `path` field points to the symlink target. Updates are skipped for linked skills (`SKIP: <name> is linked.`).

**npm install:** Runs `npm pack` + extract into the cache, then copies to the skills dir. Pinning via `@version` suffix controls the exact package version.
