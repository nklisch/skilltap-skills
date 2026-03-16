# skilltap — Human guide

Use this reference when the user wants to understand skilltap, set it up for the first time, or use interactive/human-mode features. Walk them through the relevant sections.

## What is skilltap?

skilltap is a CLI tool for managing agent skills — reusable instruction bundles that teach AI agents (Claude Code, Cursor, Codex, Gemini, Windsurf) how to perform specific tasks. It handles installing from git repos, npm, or curated registries (taps), with built-in security scanning.

## First-time setup

Guide the user through these steps in order:

1. **Install skilltap** (if not installed):
   ```bash
   npm install -g skilltap
   ```

2. **Run the setup wizard**:
   ```bash
   skilltap config
   ```
   This interactively configures: default scope (global vs project), agent symlink targets, and security settings.

3. **Enable agent mode** (so agents can use skilltap non-interactively):
   ```bash
   skilltap config agent-mode
   ```
   This sets up: agent-mode enabled flag, default scope for agent operations, and agent-specific security policy.

4. **Add a tap** (optional, for curated skill discovery):
   ```bash
   skilltap tap add skilltap https://github.com/nklisch/skilltap-skills
   ```

## Security configuration

Security has separate settings for human and agent modes. Humans configure via interactive wizard:

```bash
skilltap config security
```

This walks through:
- Which mode to configure (human, agent, or both)
- Preset selection (none, relaxed, standard, strict)
- Custom options if no preset fits (scan level, warning behavior, require-scan)
- Trust overrides for specific taps or source types

### Presets explained

| Preset | What it does |
|--------|-------------|
| **none** | No scanning at all — trust everything |
| **relaxed** | Static scan runs but warnings are ignored |
| **standard** | Static scan, prompts on warnings (recommended for humans) |
| **strict** | Semantic AI scan, blocks on any concern (recommended for agents) |

### Trust overrides

Allow relaxing or tightening security for specific sources:
- Trust a known tap: `skilltap config security --trust tap:home=relaxed`
- Be strict with npm: `skilltap config security --trust source:npm=strict`

## Interactive commands

These commands use interactive UI (prompts, spinners, colored output) and are only available in human mode:

- `skilltap config` — full setup wizard
- `skilltap config agent-mode` — agent mode toggle
- `skilltap config security` — security wizard
- `skilltap config telemetry` — telemetry toggle
- `skilltap find --interactive` / `skilltap find -i` — type-ahead skill search with install picker
- `skilltap skills adopt` (without names) — multiselect picker for unmanaged skills

## Understanding skill scopes

- **Global**: installed in `~/.agents/skills/`, available everywhere
- **Project**: installed in `./.agents/skills/`, available in the current repo only
- **Linked**: symlinked from a local directory (for development)

## Agent symlinks (`--also`)

When installing a skill, `--also <agent>` creates symlinks in agent-specific directories so multiple agents can discover the skill:

```bash
skilltap install user/repo --also claude-code --also cursor
```

Supported agents: `claude-code`, `cursor`, `codex`, `gemini`, `windsurf`

## Disabled vs removed skills

- **Disable**: hides the skill from agents temporarily — files move to `.agents/skills/.disabled/` but remain installed. Re-enable anytime.
- **Remove**: permanently deletes the skill and all symlinks.

## Adopting unmanaged skills

If skills were manually placed in agent directories (not installed via skilltap), the adopt command brings them under management:

```bash
skilltap skills adopt              # interactive picker
skilltap skills adopt my-skill     # by name
skilltap skills adopt my-skill --track-in-place  # don't move, just track
```
