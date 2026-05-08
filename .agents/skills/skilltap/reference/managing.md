# Managing Skills and Plugins

## Listing skills

```bash
skilltap skills [--project] [--global] [--unmanaged] [--disabled] [--active] [--json]
```

Output groups:
- **managed** — installed via `skilltap install`, tracked in `installed.json`
- **linked** — local symlinks installed via `skilltap skills link`
- **unmanaged** — skill directories found in `.agents/skills/` not tracked by skilltap

Flags:
| Flag | Effect |
|---|---|
| `--project` | Project scope only |
| `--global` | Global scope only |
| `--unmanaged` | Only unmanaged skills |
| `--disabled` | Only disabled skills |
| `--active` | Only active skills |
| `--json` | JSON output |

## Skill info

```bash
skilltap skills info <name> [--json]
```

Returns the full record from `installed.json` for a managed skill. Fields: `name`, `description`, `repo`, `ref`, `sha`, `scope`, `path`, `tap`, `also`, `installedAt`, `updatedAt`, `trust`, `active`. See [state-files.md](state-files.md) for field-by-field types.

## Disabling and enabling skills

```bash
skilltap skills disable <name>
skilltap skills enable <name>
```

**Managed skills:** The skill directory is physically moved to/from `.agents/skills/.disabled/<name>/`. All agent symlinks are updated accordingly.

**Linked skills:** The `active` flag in `installed.json` is toggled; the symlink in the agent dir is created or removed.

Both commands update any `--also` symlinks automatically.

## Removing skills

```bash
skilltap skills remove [name...] [--yes]
```

In agent mode, `--yes` is auto-applied — no confirmation prompt.

Works on managed, linked, and **unmanaged** skills. For unmanaged skills, the directory is deleted from `.agents/skills/`. For linked skills, the symlink is removed (the original directory is not deleted).

**Important:** Do NOT use `skilltap skills remove` on a plugin-owned skill. Plugin skills are not in `installed.json` — the command will no-op or fail. Use `skilltap plugin remove` or `skilltap plugin toggle` instead.

## Adopting unmanaged skills

```bash
skilltap skills adopt [name...] [--track-in-place]
```

Brings unmanaged skills (found in `.agents/skills/` but not in `installed.json`) under skilltap management. If `name` is omitted, prompts interactively (in agent mode: auto-selects all).

`--track-in-place`: Creates a `linked`-scope record pointing to the current location instead of moving the directory.

Without `--track-in-place`: The skill stays where it is and is recorded with `scope: project` or `scope: global` depending on which `.agents/skills/` directory it was found in.

## Moving skills between scopes

```bash
skilltap skills move <name> --project
skilltap skills move <name> --global
```

Exactly one of `--project` or `--global` must be specified. The skill directory is moved and all agent symlinks are updated. The `installed.json` record is updated in both scopes (removed from source, added to destination).

## Linking and unlinking local skills

```bash
skilltap skills link <path> [--also <agent>] [--project] [--global]
skilltap skills unlink <name>
```

`link`: Creates a symlink in `.agents/skills/<name>` pointing to the absolute path of `<path>`. The skill directory is not copied. Record is added with `scope: linked`.

`unlink`: Removes the symlink and the `installed.json` record. The original directory is not deleted.

## Updating skills

```bash
skilltap update [name] [--yes] [--strict] [--semantic] [--check] [--json]
```

- Omit `name` to update all installed skills.
- Linked skills are skipped (`SKIP: <name> is linked.`).
- Disabled skills are skipped (`SKIP: <name> — disabled`).
- If the skill is already at the latest commit, output is `OK: <name> is already up to date.`
- `--check` / `-c`: Refresh the update cache and report available updates without applying them.

Update output line format:
```
OK: Updated <name> (<from-sha> -> <to-sha>)
OK: <name> is already up to date.
SKIP: <name> — <reason>
```

## Plugin listing

```bash
skilltap plugin [--global] [--project] [--json]
```

Agent-mode plain-text format:
```
GLOBAL <name> 3 skills, 2 MCPs, 1 agent source=<repo>
PROJECT <name> 1 skill, 0 MCPs, 0 agents source=<tap>
```

`--json` returns the full plugin records from `plugins.json`.

## Plugin info

```bash
skilltap plugin info <name> [--json]
```

Returns the full plugin record including all components with their `active` state. Use this before toggling to determine current state, since `plugin toggle` flips individual states rather than setting them.

## Plugin toggle

```bash
skilltap plugin toggle <name> [--skills] [--mcps] [--agents] [--json]
```

**In agent mode, at least one category flag is required.** Without `--skills`, `--mcps`, or `--agents`, the command exits 1 with:
```
Provide --skills, --mcps, or --agents to specify what to toggle.
```

Each targeted component's `active` state is **flipped individually** (active → disabled, disabled → active). There is no "set all to enabled" mode. To force a specific state:

1. Call `skilltap plugin info <name> --json` to check current state.
2. Only toggle the components that are not already in the desired state.

Toggling skills: moves skill directories to/from `.disabled/`, updates symlinks.
Toggling MCPs: injects or removes MCP server entries from agent config files.
Toggling agents: moves agent definition files to/from `.disabled/`, or writes/removes them.

## Plugin remove

```bash
skilltap plugin remove <name> [--yes] [--json]
```

In agent mode, `--yes` is auto-applied.

Removes the plugin and all its components:
- Skill directories deleted from `.agents/skills/`
- MCP entries removed from agent config files (only `skilltap:`-namespaced entries)
- Agent definition files deleted from `.claude/agents/`
- Record removed from `plugins.json`

User-added MCP entries (not namespaced with `skilltap:`) are not touched.

Output:
```
OK: Removed plugin <name> (3 skills, 2 MCPs, 1 agent)
```

## Important rules

- Plugin-owned skills do not appear in `installed.json` — they are in `plugins.json` only. Never try to manage them with `skilltap skills` commands.
- `skilltap plugin` is also accessible as `skilltap plugins` (alias, not promoted).
- The `plugin toggle` flip is per-component, not per-plugin. Running toggle twice returns to the original state.
