---
name: skilltap-find
description: Discover and search for agent skills via configured taps and the skills.sh registry. Use when the user wants to find, discover, or search for available agent skills to install.
license: MIT
---
# skilltap-find

Search for installable agent skills via taps and the skills.sh registry.

## Search

```bash
skilltap find <query> --json          # structured output for parsing
skilltap find <query>                 # plain text table
skilltap find <query> --local         # taps only, skip registry
skilltap find                         # list all tap skills (no registry search)
```

Multi-word queries work without quoting: `skilltap find git hooks`

Registry searches require a query of at least 2 characters. Results: tap matches first, then registry results sorted by install count.

## JSON output fields

`name`, `description`, `source`, `installRef`, `skill` (if multi-skill repo), `installs` (registry only)

## Install a found skill

Use the `installRef` from JSON output, or the skill name directly:

```bash
skilltap install <skill-name> --project
skilltap install <skill-name> --global
skilltap install <skill-name>@v1.2.0 --project
```

## Trust tiers

- `provenance` — SLSA attestation verified via Sigstore
- `publisher` — known publisher
- `curated` — tap maintainer manually verified
- `unverified` — no verification; review before installing

## Manage taps

Taps are git repos containing a `tap.json` registry.

```bash
skilltap tap add <name> <url>
skilltap tap remove <name>
skilltap tap list
skilltap tap update [name]
```

Add the official tap:
```bash
skilltap tap add skilltap https://github.com/nklisch/skilltap-skills
```
