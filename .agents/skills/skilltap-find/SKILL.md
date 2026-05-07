---
name: skilltap-find
description: Discover and search for installable agent skills and plugins via taps and the skills.sh registry. Use when the user wants to find, discover, or search for available agent skills or plugins to install.
license: MIT
---
# skilltap-find

Search for installable agent skills and plugins via taps and the skills.sh registry.

## Search

```bash
skilltap find <query> --json          # structured output for parsing
skilltap find <query>                 # plain text table
skilltap find <query> -l              # taps only, skip registry
skilltap find <query> --local         # taps only, skip registry (same as -l)
skilltap find                         # list all tap skills (non-TTY: table; TTY: interactive)
```

Multi-word queries work without quoting: `skilltap find git hooks`

Registry searches require a query of at least 2 characters. Results: tap matches first, then registry results sorted by install count.

`-i, --interactive` forces interactive mode (type-ahead picker) even with a query or non-TTY stdout. Interactive mode is not useful for agents — omit it.

## JSON output fields

`name`, `description`, `source`, `installRef`, `skill` (if multi-skill repo), `installs` (registry only), `plugin` (boolean, present on plugin entries)

Plugin entries have `"plugin": true` in JSON output and show a `[plugin]` badge in plain-text output.

## Install a found skill or plugin

Use the `installRef` from JSON output, or the skill/plugin name directly:

```bash
skilltap install <skill-name> --project
skilltap install <skill-name> --global
skilltap install <skill-name>@v1.2.0 --project
skilltap install <tap-name>/<plugin-name> --project   # tap-defined plugin
```

For plugins, `installRef` may be in `tap-name/plugin-name` form.

## Trust tiers

Trust signals are informational — they answer "who published this?" and do NOT block installation. Security scanning is what blocks.

- `provenance` — SLSA attestation (npm Trusted Publishing or GitHub Artifact Attestations) verified via Sigstore
- `publisher` — known but uncryptographically-verified identity
- `curated` — skill came from a tap (tap maintainer curated)
- `unverified` — no trust signal; review before installing

## Manage taps

Taps are git repos or HTTP registries containing a `tap.json` index. The `skilltap-skills` tap ships built-in — no need to add it manually.

```bash
skilltap tap add <name> <url> [--type git|http]   # add a tap (auto-detects git vs HTTP)
skilltap tap add <owner/repo>                      # GitHub shorthand
skilltap tap remove <name> [--yes]                 # --yes skips confirmation
skilltap tap list [--json]
skilltap tap update [name]
skilltap tap info <name> [--json]                  # name, type, url, path, last fetched, skill count
```

`tap info` type field values: `git`, `http`, `builtin`.

`tap install` and `tap init` are interactive-only (human mode); not for agents.

### HTTP registry taps

Taps can be HTTP registries instead of git repos. `tap add` auto-detects based on URL. HTTP taps are always live (no local clone); `tap update` skips them. Bearer-token auth for private registries is configured in `config.toml` (`auth_token` or `auth_env`), not via flags. For `find`, HTTP taps work identically to git taps.

### Tap-defined plugins

Taps can define plugins inline in `tap.json`. Install with `skilltap install <tap-name>/<plugin-name>`. Plugin entries appear with `[plugin]` badge in `find` results.
