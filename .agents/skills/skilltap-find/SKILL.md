---
name: skilltap-find
description: "find / search / discover / look up / browse / list available agent skills, plugins, and standalone MCP servers via skilltap taps and the skills.sh registry. Triggers when the user wants to find something installable or browse what's available."
license: MIT
---

# skilltap-find

Load this skill when the user asks to find, search, discover, browse, or list installable agent skills, plugins, or MCP servers. Covers the `skilltap find` command, tap management, install source formats, and trust tiers. After picking a result, hand off to `../skilltap/SKILL.md` to install it.

## Quickstart

```bash
skilltap find <query> --json
```

Multi-word queries work without quoting: `skilltap find git hooks` joins the args.

Parse the JSON array and pick the best match:

```json
[
  {
    "name": "commit-helper",
    "description": "Conventional commit messages",
    "source": "home",
    "installRef": "commit-helper",
    "skill": "commit-helper",
    "installs": 18443
  },
  {
    "name": "vercel-react-bp",
    "description": "React best practices",
    "source": "skills.sh",
    "installRef": "vercel/react-best-practices",
    "installs": 184500
  }
]
```

Field meanings:
- `source` — tap name (e.g. `home`, `skilltap-skills`) or `"skills.sh"` (registry hit).
- `installRef` — pass this verbatim to `skilltap install`. Do NOT construct it yourself.
- `skill` — pre-selected skill name for multi-skill repos (optional).
- `plugin` — `true` if the entry is a tap-defined plugin.
- `installs` — present on registry results only.

## Decision rules

1. Run `skilltap find <query> --json`.
2. If no results: try a shorter or broader query. Offer to add a tap (`skilltap tap add`) if the user has none configured.
3. Pick the best match: prefer tap results (local, curated) over registry; break ties by `installs` count.
4. Pass `installRef` to the right typed install verb:
   - Skill entry → `skilltap install skill <installRef>`
   - Plugin entry (tap-defined plugin or `entry.plugin === true`) → `skilltap install plugin <installRef>`
   - MCP entry → `skilltap install mcp <installRef>`

   See `../skilltap/SKILL.md` for the full install flow (smart-scope, `--also`, security policy).

## Flags

| Flag | Short | Effect |
|---|---|---|
| `--json` | | Machine-readable array output. Auto-selected when stdout is non-TTY. |
| `--local` | `-l` | Skip the skills.sh registry; tap results only. |
| `--interactive` | `-i` | Type-ahead picker (HUMAN ONLY — requires TTY). |

Notes:
- Registry search requires ≥ 2 characters in the query.
- Non-TTY + no taps + no query → empty-state message (not an error).
- Result order: tap matches first, then registry sorted by install count descending.
- The TUI dashboard (`skilltap` bare invocation in a TTY) has a Find tab that wraps the same search.

## Tap management (quick reference)

| Command | Notes |
|---|---|
| `skilltap tap add <name> <url>` | Add a git tap. URL is treated as git only — no HTTP fallback. |
| `skilltap tap add <owner/repo>` | GitHub shorthand — derives name and URL from `default_git_host`. |
| `skilltap tap remove <name> [--yes]` | Remove a tap. |
| `skilltap tap list [--json]` | List configured taps. |
| `skilltap tap info <name> [--json]` | Name, type, URL, path, last fetched, skill count. |
| `skilltap tap init <directory>` | Scaffold a new tap repo — HUMAN-friendly, can be run anywhere. |

The built-in `skilltap-skills` tap is pre-configured and searched automatically — do not add it manually with `tap add`. Disable it with `builtin_tap = false` in `config.toml`. It's lazily cloned on first use to `~/.config/skilltap/taps/skilltap-skills/`.

There is **no `skilltap tap update` command**. Tap repos are refreshed on use (during `find` or `install`) when their cached clone is stale; `skilltap doctor` reports unreachable taps.

## Reference pages

- `reference/sources.md` — every accepted `skilltap install <type> <source>` form, resolution order, and per-form gotchas.
- `reference/taps.md` — tap concepts, `tap.json` schema, scaffold + maintenance workflow, marketplace.json fallback.
- `reference/trust.md` — trust tier semantics, what each tier signals, why trust never blocks an install.
