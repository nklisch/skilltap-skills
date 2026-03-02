---
name: skilltap-find
description: Discover and search for agent skills via configured taps and the npm registry.
license: MIT
---
# skilltap-find

Discover installable agent skills using `skilltap find` (searches configured taps) or `skilltap find --npm` (searches npm).

## Search taps

```bash
skilltap find                   # list all skills in all configured taps
skilltap find <query>           # filter by name/description/tags
skilltap find <query> --json    # machine-readable output
skilltap find <query> -i        # interactive fuzzy finder (TTY only)
```

Output columns: `tap`, `name`, `description`, `trust`

## Search npm

```bash
skilltap find --npm                 # list all npm packages tagged agent-skill
skilltap find --npm <query>         # filter by query
skilltap find --npm <query> --json
```

npm results include package name, description, version, and trust tier.

## Trust tiers

| Tier | Meaning |
|------|---------|
| `provenance` | SLSA attestation verified via Sigstore — package built from a specific GitHub Actions run |
| `publisher` | Known npm publisher (npmUser match) |
| `curated` | Tap maintainer has manually verified the skill |
| `unverified` | No verification — review before installing |

Prefer `provenance` or `curated` skills for production use.

## Manage taps

Taps are git repos containing a `tap.json` skill registry.

```bash
skilltap tap list                        # list configured taps
skilltap tap add <name> <url>            # add a tap (clones repo)
skilltap tap remove <name>               # remove a tap
skilltap tap update                      # update all taps
skilltap tap update <name>               # update one tap
```

### Add a tap

```bash
skilltap tap add skilltap https://github.com/nklisch/skilltap-skills
```

After adding, skills in that tap become searchable via `skilltap find`.

## Install a found skill

Once you've identified a skill, install it by tap name (no repo URL needed):

```bash
skilltap install <skill-name>              # prompts for scope
skilltap install <skill-name> --global --yes
skilltap install <skill-name> --project --yes
skilltap install <skill-name>@v1.2.0      # pin to version/ref
```

Source resolution tries tap names before GitHub shorthand, so bare names work after `tap add`.
