---
name: skilltap-find
description: "find / search / discover / look up / browse / list available agent skills and plugins via skilltap taps and the skills.sh registry."
license: MIT
---

# skilltap-find

Load this skill when the user asks to find, search, discover, browse, or list installable agent skills or plugins. This skill covers the `skilltap find` command, tap management, install source formats, and trust tiers. After selecting a result, hand off to `../skilltap/SKILL.md` to install it.

## Quickstart

```bash
skilltap find <query> --json
```

Multi-word queries work without quoting: `skilltap find git hooks`.

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
- `source` — tap name or `"skills.sh"` (registry hit)
- `installRef` — pass this verbatim to `skilltap install`; do NOT construct it yourself
- `skill` — pre-selected skill name for multi-skill repos (optional)
- `installs` — present on registry results only

## Decision rules

1. Run `skilltap find <query> --json`.
2. If no results: try a shorter or broader query. Offer to add a tap (`skilltap tap add`) if the user has none.
3. Pick the best match: prefer tap results (local, curated) over registry; break ties by `installs` count.
4. Pass `installRef` to `skilltap install` — see `../skilltap/SKILL.md` for the full install flow.

## Flags

| Flag | Short | Effect |
|------|-------|--------|
| `--json` | | Machine-readable array output — use in agent mode |
| `--local` | `-l` | Skip skills.sh registry; tap results only |
| `--interactive` | `-i` | Type-ahead picker (HUMAN ONLY — requires TTY) |

Notes:
- Registry search requires >= 2 characters in the query.
- Non-TTY + no taps + no query → empty-state message (not an error).
- Results order: tap matches first, then registry sorted by install count descending.

## Tap management (quick reference)

| Command | Notes |
|---------|-------|
| `skilltap tap add <name> <url>` | Add a git or HTTP registry tap |
| `skilltap tap add <owner/repo>` | GitHub shorthand — derives name and URL automatically |
| `skilltap tap remove <name>` | Remove a tap (`--yes` to skip prompt) |
| `skilltap tap list [--json]` | List configured taps |
| `skilltap tap update [name]` | Re-fetch tap index (git: pull; http: refresh cache) |
| `skilltap tap info <name> [--json]` | Name, type, URL, path, last fetched, skill count |
| `skilltap tap install` | Multiselect picker — HUMAN ONLY |
| `skilltap tap init <name>` | Scaffold a new tap repo — HUMAN ONLY |

The built-in `skilltap-skills` tap is pre-configured and searched automatically — do not add it manually. Disable with `builtin_tap = false` in config.

## Reference pages

- `reference/sources.md` — every accepted `skilltap install <source>` format, resolution order, and per-form gotchas.
- `reference/taps.md` — tap concepts, git vs. HTTP types, `tap.json` schema, auth for private registries.
- `reference/trust.md` — trust tier semantics, what they signal, what they don't (trust never blocks installs).
