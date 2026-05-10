# Managing Skills, Plugins, and MCP Servers

Post-install management is split across a small set of typed verbs at the top level. There is no `skilltap skills` or `skilltap plugin` subgroup — operations live directly on the root command (`info`, `update`, `remove`, `toggle`, `adopt`, `move`).

## Listing: `status`

```bash
skilltap status [--json] [--unmanaged] [--disabled] [--active] [--global] [--project]
```

Headless equivalent of the bare `skilltap` TUI dashboard. Lists all installed skills, plugins, and MCP servers grouped by scope, with source attribution.

| Flag | Effect |
|---|---|
| `--global` | Boolean — include only global-scope items. |
| `--project` | Boolean — include only project-scope items. |
| `--unmanaged` | Show skill directories on disk but not tracked in `state.json`. |
| `--disabled` | Only items with `active: false`. |
| `--active` | Only items with `active: true`. |
| `--json` | Machine-readable output (auto-selected when stdout is non-TTY). |

```
$ skilltap status

Global (.agents/skills/) — 3 skills
  commit-helper   managed   claude-code   nklisch/skills
  code-review     managed   claude-code   nklisch/skills
  my-skill        managed   —             ~/dev/my-skill (adopted)

Project (.agents/skills/) — 1 skill
  bun             managed   claude-code   nklisch/skills

Plugins — 1
  dev-toolkit     managed   3 skills, 2 MCPs, 1 agent   corp/dev-toolkit

Taps — 2
  home            https://gitea.example.com/nathan/my-skills-tap   4 skills
  skilltap-skills (built-in)                                       47 skills
```

`status` is in the `SKIP_STARTUP_ARGS` list — it does not run the self-update or skill-update checks, making it safe and fast for repeated polling.

## Per-item details: `info`

```bash
skilltap info <name> [--global] [--project] [--json]
```

Show the full record from `state.json` for a managed skill, plugin, or standalone MCP server. Use `--global` / `--project` to disambiguate when the same name exists in both scopes.

```
$ skilltap info dev-toolkit

dev-toolkit (plugin, global)
  Developer workflow toolkit
  Source: corp/dev-toolkit
  Components:
    [skill]  code-review     active
    [skill]  commit-helper   active
    [skill]  test-generator  disabled
    [mcp]    database        active
    [mcp]    file-search     disabled
    [agent]  code-review     active
```

For schema field details, see [state-files.md](state-files.md).

## Updating: `update`

```bash
skilltap update                              # update everything
skilltap update skill                        # update all skills
skilltap update plugin                       # update all plugins
skilltap update mcp                          # update all standalone MCPs
skilltap update skill <name>                 # update one
skilltap update plugin <name>
skilltap update mcp <name>
```

| Flag | Effect |
|---|---|
| `--scope project\|global` | Restrict to one scope. |
| `--strict` | Skip items whose diff has security warnings. |
| `--semantic` | Force Layer 2 semantic scan on every diff. |
| `--skip-scan` | Skip diff scanning. |
| `--check`, `-c` | Refresh the update cache and report available updates without applying. |
| `--force`, `-f` | Force update even if already at latest SHA. |
| `--yes` | Auto-accept clean updates (security warnings still gated by `on_warn`). |
| `--quiet` | Suppress per-step details. |
| `--json` | Machine-readable output. |

For each target: fetch → diff → scan → confirm → pull. Linked skills are skipped (`SKIP: <name> is linked.`). Disabled skills are skipped (`SKIP: <name> — disabled`).

Output line format:
```
OK: Updated <name> (<from-sha> -> <to-sha>)
OK: <name> is already up to date.
SKIP: <name> — <reason>
```

Final summary:
```
Updated: 2   Skipped: 1   Up to date: 4
```

`update` honors the project manifest+lockfile when project scope is in play: pulling a new ref refreshes `skilltap.lock` to match.

## Removing: `remove`

```bash
skilltap remove skill   <name> [--scope project|global] [--yes] [--json]
skilltap remove plugin  <name> [--scope project|global] [--yes] [--json]
skilltap remove mcp     <name> [--scope project|global] [--yes] [--json]
```

In TTY mode, prompts to confirm. `--yes` skips the prompt.

`remove plugin <name>` removes the plugin record and **all** components: skill directories deleted, MCP entries removed from agent config files (only `skilltap:`-namespaced ones), agent definition files deleted, lockfile entry dropped.

**Plugin-component skills:** `remove skill <name>` on a skill that is a plugin component errors with a hint pointing at `remove plugin <name>` (or `toggle plugin <name>:<component>` to disable just one component without removal).

**Standalone MCP:** `remove mcp <name>` removes the entry from `state.mcpServers[]` and prunes the corresponding entries from each per-agent config file.

User-added MCP entries (not namespaced with `skilltap:`) are never touched by `remove`.

```
$ skilltap remove plugin dev-toolkit --yes
OK: Removed plugin dev-toolkit (3 skills, 2 MCPs, 1 agent)
```

## Toggling active state: `toggle`

```bash
skilltap toggle                              # TUI: pick type → name → component(s)
skilltap toggle skill <name>                 # toggle whole skill
skilltap toggle plugin <name>                # TUI scoped to plugin's components
skilltap toggle plugin <name>:<component>    # toggle ONE component (no TUI)
skilltap toggle mcp <name>                   # toggle whole MCP
```

`toggle` flips the `active` state. There is no separate `enable` / `disable`. Run `info <name>` first if you need to know current state before flipping.

| Form | What flips |
|---|---|
| `toggle skill <name>` | Whole skill — directory moves to/from `.disabled/`, agent symlinks recreated/removed. |
| `toggle plugin <name>` | TTY only. Multi-select picker over the plugin's components. |
| `toggle plugin <name>:<component>` | Just that one component. |
| `toggle mcp <name>` | Standalone MCP — entry injected/removed from each `--also` agent config. |

Only `plugin` accepts the `:component` suffix. `:component` is the component name from `state.plugins[].components[].name`.

Component-specific toggle effects:
- **skill component**: directory moved to/from `.disabled/`, symlinks recreated/removed.
- **mcp component**: entry injected into / removed from each agent's config file.
- **agent component**: file moved to/from `.claude/agents/.disabled/<name>.md`.

```
$ skilltap toggle plugin dev-toolkit:test-generator
OK: Disabled component: test-generator (skill)
```

## Adopting unmanaged content: `adopt`

```bash
skilltap adopt                       # TUI picker over all unmanaged content
skilltap adopt <path>                # adopt skill at external path (track-in-place)
skilltap adopt <path> --move         # move dir into canonical location
skilltap adopt <name>                # adopt named unmanaged item
skilltap adopt --source claude-code  # picker scoped to one agent source
```

Replaces the v0.x `link` / `unlink` verbs and consolidates externally-managed plugin adoption (Claude Code marketplace plugins, etc.) into one verb.

| Flag | Effect |
|---|---|
| `--scope project\|global` | Smart-default applies otherwise. |
| `--source <agent>` | Filter the picker to one source (e.g. `claude-code`). |
| `--also <agent>` | Add per-agent symlinks. |
| `--move` | Move the directory into the canonical install dir; default is track-in-place. |
| `--skip-scan` | Skip security scan. |
| `--yes` | Auto-accept all prompts. |
| `--json` | Machine-readable output. |

**Track-in-place (default):** symlinks the external dir into `.agents/skills/<name>`, records as `scope: linked` with `path: <external>`. The original directory is untouched on `remove`.

**With `--move`:** moves the dir into the canonical location, leaves a symlink at the old path so existing references still work.

```
$ skilltap adopt ~/dev/my-skill
OK: Adopted my-skill (tracked in-place: ~/.agents/skills/my-skill → ~/dev/my-skill)

$ skilltap adopt ~/dev/my-skill --move
OK: Moved my-skill to ~/.agents/skills/my-skill (symlink left at ~/dev/my-skill)
```

Bare `adopt` (no path) requires a TTY for the multi-select picker. In non-interactive contexts, pass a path or `--source <agent>`.

## Moving between scopes: `move`

```bash
skilltap move <name> --scope project|global [--also <agent>] [--yes] [--json]
```

Move a managed skill from one scope to the other. The directory is moved on disk, all `--also` symlinks are re-created in the new scope, the `state.json` record is removed from the source scope and added to the destination, and the project manifest+lockfile is updated when project scope is involved.

Linked skills cannot be moved — adopt them with `--move` first if you want to convert.

```
$ skilltap move patterns --scope global
OK: Moved patterns: .agents/skills/patterns → ~/.agents/skills/patterns
```

## Previewing without installing: `try`

```bash
skilltap try skill   <source> [--skip-scan] [--json]
skilltap try plugin  <source> [--skip-scan] [--json]
skilltap try mcp     <source> [--skip-scan] [--json]
```

Read-only preview. Clones (or copies, for local paths) the source to a temp directory, parses any manifests, displays the structure, runs the static security scan, prints SKILL.md or plugin manifest contents. Never writes to install paths or state — cleans up after itself.

Useful for vetting a source before installing, inspecting a multi-skill repo's contents, or scanning a tap-defined plugin without committing.

```
$ skilltap try plugin corp/dev-toolkit

Cloning corp/dev-toolkit...
Detected plugin: dev-toolkit
  3 skills, 2 MCP servers, 1 agent definition
  Skills: code-review, commit-helper, test-generator
  MCPs: database (stdio), file-search (stdio)
  Agents: code-review.md

✓ No security warnings

Run 'skilltap install plugin corp/dev-toolkit' to install.
```

## Reconciling project state: `sync`

See [manifest.md](manifest.md) for the full `skilltap.toml` + `skilltap.lock` workflow.

```bash
skilltap sync [--apply] [--strict] [--json]
```

Reconciles three sources of truth: `skilltap.toml` (manifest, committed), `skilltap.lock` (lockfile, committed), `state.json` (local on-disk state). Without `--apply`, prints a drift report grouped by kind. With `--apply`, executes the diff via `install` / `remove`.

Requires a project root (a directory containing `skilltap.toml` or `.git`). Errors otherwise with `skilltap sync requires a project root`.

## Health checks: `doctor`

```bash
skilltap doctor [--fix] [--json]
```

Checks environment, config, state, manifest+lockfile drift, symlink integrity, tap reachability, MCP injection consistency, plugin manifest schemas, and v1-state orphans. `--fix` auto-repairs safe issues (broken symlinks, orphan records, corrupt manifests).

```
$ skilltap doctor

  ✓ git: /usr/bin/git (2.53.0)
  ✓ config: /home/user/.config/skilltap/config.toml
  ✓ dirs: /home/user/.config/skilltap
  ✓ installed: 7 skills (3 global, 4 project)
  ⚠ symlinks: 1 broken
  ✓ taps: 2 reachable
  ✓ MCP injection consistent
```

`--fix` exits 0 when all fixes succeed; only non-fixable failures cause exit 1.

`doctor` is also the per-artifact validator (replaces v0.x `verify`):

```bash
skilltap doctor skill   <path>   # SKILL.md exists, frontmatter valid, name matches dir, scan clean
skilltap doctor plugin  <path>   # manifest schema valid, all references resolve, name matches dir
```

## Important rules

- **Plugin-owned skills are not in `state.skills[]`.** They live in `state.plugins[].components[]`. Use plugin commands or `toggle plugin <name>:<component>`.
- **`toggle` flips, it does not set.** Two `toggle` invocations return to the original state. To enforce a specific state, check `info <name> --json` first.
- **`adopt` defaults to track-in-place.** Pass `--move` if you want to relocate the directory.
- **`status`, `info` use `--global`/`--project` as boolean filters.** `install`, `remove`, `update`, `move` use `--scope <value>`.
- **`info`, `move`, `adopt` are top-level commands** — no `skilltap skills` or `skilltap plugin` subgroup exists.
