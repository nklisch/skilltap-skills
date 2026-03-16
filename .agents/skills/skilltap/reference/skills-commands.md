# Skills command group

Unified view and management of all installed skills.

## Contents
- List skills
- Skill info
- Disable / enable
- Adopt unmanaged skills
- Move between scopes
- Remove

## List skills

```bash
skilltap skills                       # table: name, status, agents, source
skilltap skills --json                # JSON array
skilltap skills --global              # global scope only
skilltap skills --project             # project scope only
skilltap skills --unmanaged           # unmanaged skills only
skilltap skills --disabled            # disabled skills only
skilltap skills --active              # active skills only
```

Skills are grouped by scope (global vs project) and management status (managed/linked vs unmanaged).

## Skill info

```bash
skilltap skills info <name>           # source, path, SHA, trust tier, symlinks
```

## Disable / enable

Temporarily hide a skill from agents without removing it.

```bash
skilltap skills disable <name> [--global|--project]
skilltap skills enable <name> [--global|--project]
```

Behavior:
- **Managed skills**: moves between `.agents/skills/{name}` and `.agents/skills/.disabled/{name}`
- **Linked skills**: marks as inactive in installed.json (no move)
- Both: removes/recreates agent symlinks and sets `active` flag

## Adopt unmanaged skills

Bring skills found in agent-specific directories under skilltap management.

```bash
skilltap skills adopt [name...] [--global|--project] [--track-in-place] [--also <agent>] [--skip-scan] [--yes]
```

- Default mode: moves skill to `.agents/skills/{name}` and creates symlinks back
- `--track-in-place`: tracks at current location instead of moving
- Runs security scanning unless `--skip-scan` is passed
- Interactive multiselect when no names provided (human mode only)

## Move between scopes

```bash
skilltap skills move <name> --global|--project [--also <agent>]
```

Must specify exactly one of `--global` or `--project`. Moving to project scope requires being inside a project directory.

## Remove

```bash
skilltap skills remove <name> [--global|--project]
```

No confirmation prompt in agent mode.
