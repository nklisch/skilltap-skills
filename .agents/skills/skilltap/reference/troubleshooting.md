# Troubleshooting

## "Legacy config detected — run 'skilltap migrate' to upgrade"

**Symptom:** Any command fails with `Legacy config detected — run 'skilltap migrate' to upgrade` (or a soft hint: `↑  Legacy state detected. Run 'skilltap migrate' to upgrade.`).

**Cause:** A pre-v2 `config.toml`, `installed.json`, or `plugins.json` exists. `loadConfig` hard-fails on legacy schema markers (per-mode `[security]`, `[agent-mode]`, `preset = ...`, `require_scan`, etc.).

**Fix:** Run `skilltap migrate` once on this machine. It collapses per-mode `[security.human]` / `[security.agent]` blocks into a flat `[security]` block (stricter values win), extracts operational keys to `[scanner]`, drops `[agent-mode]`, and consolidates `installed.json` + `plugins.json` into `state.json`. Originals are renamed to `*.v1.bak`.

```bash
skilltap migrate
```

If `migrate` reports HTTP taps in your config, they need to be removed or converted manually before migration completes — skilltap is git-only for taps as of v2.

## "skilltap requires a TTY for the dashboard"

**Symptom:** Bare `skilltap` invocation prints this error and exits 1.

**Cause:** Bare `skilltap` (no subcommand) opens the Ink-based TUI. Without a TTY (piped, redirected, agent-driven), it refuses.

**Fix:** Use `skilltap status [--json]` for headless output:

```bash
skilltap status              # plain text dashboard
skilltap status --json       # machine-readable
```

## SECURITY ISSUE FOUND / "--strict" warning block

**Symptom:** Install fails with a warning block listing file:line, raw + visible content, decoded base64, etc., followed by `error: Security warnings found (strict mode). Aborting install.` or `Security warnings found and stdin is not a TTY (cannot prompt).`

**Rules when an agent sees this:**

1. **Stop.** Exit 1 is intentional.
2. **Relay the entire warning block verbatim** to the user. Do not summarize.
3. **Do not retry with `--skip-scan`** without the user explicitly authorizing it.
4. **Do not advise disabling security settings** as a default workaround.

The user must inspect the flagged file and lines manually. To install anyway after review, they can pass `--skip-scan` themselves, set `[security].on_warn = "install"` (default), or add a matching glob to `[security].trust`.

## "skill 'X' not found"

**Symptom:** `info <name>` or `remove skill <name>` reports the skill is not found.

**Diagnosis:**
1. Run `skilltap status --json` to list everything and check spelling.
2. Check both scopes: `skilltap status --global` and `skilltap status --project` separately — the skill may be in the other one.
3. Check if the skill is disabled: `skilltap status --disabled`.
4. Check if it's plugin-owned: `skilltap status --json` and look for the name under `plugins[].components[]`. If so, it is NOT in `state.skills[]` — use plugin commands.

**Fix:** Pass `--global` / `--project` to target the right scope, or use plugin commands for plugin components.

## "Cannot remove: skill is a plugin component"

**Symptom:** `remove skill <name>` errors with this message.

**Cause:** The skill is owned by a plugin (lives in `state.plugins[].components[]`, not `state.skills[]`).

**Fix:**
- To remove the entire plugin: `skilltap remove plugin <plugin-name>`.
- To disable just the one skill component: `skilltap toggle plugin <plugin-name>:<skill-name>`.

Use `skilltap info <plugin-name>` to find the right plugin and component name.

## Skill installed but agent can't find it

**Symptom:** `skilltap status` lists the skill, but the agent (Claude Code, Cursor, etc.) does not load it.

**Cause:** The per-agent symlink is missing. skilltap installs to `.agents/skills/` canonically; per-agent dirs (`.claude/skills/`, `.cursor/skills/`) only get a symlink if `--also <agent>` was passed at install time (or `[defaults] also = [...]` is set in config).

**Diagnosis:** `skilltap doctor` reports broken or missing per-agent symlinks.

**Fix:** Re-run install with `--also <agent>`:

```bash
skilltap install skill <source> --also claude-code
```

Or repair via doctor:

```bash
skilltap doctor --fix
```

## state.json parse failure

**Symptom:** Any skilltap command fails with a JSON parse error mentioning `state.json`.

**Diagnosis:** The file is malformed (most commonly from a hand edit).

**Fix:**
1. Run `skilltap doctor` — it will report the exact parse error and the file path.
2. Inspect and fix the JSON at the path reported by doctor (`~/.config/skilltap/state.json` for global, `<project>/.agents/state.json` for project).
3. Re-run `skilltap doctor` to confirm.

The `version` field must remain `2`. A reasonable empty-state recovery file is:

```json
{ "version": 2, "skills": [], "plugins": [], "mcpServers": [] }
```

## "skilltap.toml is corrupt"

**Symptom:** `install --scope project` fails with `skilltap.toml is corrupt: <details>` and a hint to run `skilltap doctor --fix`.

**Behavior by output mode:**
- **TTY:** install backs up the corrupt file to `skilltap.toml.bak`, resets to empty, logs the recovery, then proceeds.
- **Non-TTY:** install refuses and exits 1. No side effects.

**Fix (non-TTY):** run `skilltap doctor --fix` to back up + reset, then retry the install.

## MCP server did not appear in agent

**Symptom:** A plugin (or standalone MCP) was installed but the server is not showing up in the agent's MCP list.

**Diagnosis:**
1. Run `skilltap info <name> --json` and check the component's `active` field.
2. If `active: false`, the component is toggled off.
3. If `active: true`, check the agent's config file directly (`~/.claude/settings.json`, `~/.cursor/mcp.json`, etc.) for the namespaced key `skilltap:<plugin>:<server>` or `skilltap:standalone:<name>`.
4. Run `skilltap doctor` — the "MCP injection consistent" check verifies that every active MCP record has a matching agent-config entry.

**Fix (toggle case):**
```bash
skilltap toggle plugin <name>:<server-name>     # one component
# or
skilltap toggle plugin <name> --mcps            # all MCPs in the plugin (TTY only)
```

## Modified agent config and want to recover pre-skilltap state

**Symptom:** After plugin / MCP install, `~/.claude/settings.json` (or equivalent) was modified and the user wants to restore the original.

**Recovery:** Before the first plugin / MCP install modified any config file, skilltap wrote a backup:

```
~/.claude/settings.json.skilltap.bak
```

Restore it:

```bash
cp ~/.claude/settings.json.skilltap.bak ~/.claude/settings.json
```

Note: this restores the file to its pre-skilltap state. Any plugins or standalone MCPs that were working will stop working until reinstalled.

## "--skip-scan" rejected / ignored

**Behavior:** `--skip-scan` is accepted on `install skill` and `install plugin`. It is NOT a mutually-exclusive opposite of `--strict` — passing both is a CLI error in some commands. There is no longer a `require_scan` config key (that was a v0.x agent-mode concept).

**If `--skip-scan` is silently not skipping:** check whether the source matches a `[security].trust` glob — those skip scanning regardless. Also check `[security].scan = "none"`, which disables scanning globally.

## Plugin capture conflict (cross-source)

**Symptom:** `install plugin <source>` errors in non-interactive mode with `cross-source capture conflict: <name> is installed from <other-source>; rerun with --force-capture to override or --no-capture to install side-by-side`.

**Cause:** A plugin component shares a name with an existing standalone, but the standalone came from a different canonical source.

**Fix:**
- `--force-capture`: capture anyway (replaces the standalone with the plugin's copy). Use only when you intend to migrate the standalone into the plugin.
- `--no-capture`: skip capture; install side-by-side. The two coexist with separate records.
- In TTY mode, you'll get an interactive prompt instead of the error.

## `sync` reports drift but everything looks installed

**Diagnosis:** Run `skilltap sync` (no `--apply`) to see the drift breakdown. Common causes:

- `lock-stale`: the locked SHA differs from on-disk. Run `sync --apply` to reinstall to lockfile.
- `ref-mismatch`: the manifest range differs from the lockfile pin. Manifest is source of truth; `sync --apply` updates the lockfile.
- `lock-orphan`: lockfile entry with no manifest declaration. Drop with `sync --apply`.
- `lock-missing`: installed but no lockfile entry. Backfill with `sync --apply`.

After `sync --apply`, all three (manifest, lockfile, state) should agree. Run `skilltap doctor` to verify.

## Doctor categories

`skilltap doctor` checks:

- **git** — `git` is on PATH.
- **config** — `config.toml` loads and parses; no legacy keys.
- **dirs** — required directories exist and are accessible.
- **state** — `state.json` schema valid; v0.x state files (`installed.json`, `plugins.json`) trigger an "orphan v1 state" finding pointing at `migrate`.
- **skills** — every `state.skills[]` record has a directory on disk; every directory on disk is tracked or unmanaged.
- **symlinks** — per-agent symlinks resolve to their canonical install dirs.
- **taps** — each configured tap's git URL is reachable.
- **manifest drift** — informational; `sync` is the executor.
- **lockfile drift** — informational; `sync --apply` reconciles.
- **plugin manifests** — every plugin's manifest schema resolves.
- **MCP injection** — every active MCP record has a matching agent-config entry; no stale `skilltap:` entries.
- **capture collisions** — plugin standalones still on disk (eligible for capture).
- **Claude Code overlap** — skills installed both by skilltap and by Claude Code's own plugin system.

`skilltap doctor --fix` repairs:
- Broken symlinks (removes and recreates).
- Orphaned records (removes `state.json` entries with missing skill directories).
- Corrupt `skilltap.toml` (backs up + resets to empty).

`--fix` exits 0 when all fixes succeed; only non-fixable failures cause exit 1.
