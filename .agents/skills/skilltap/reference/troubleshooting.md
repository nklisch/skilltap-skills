# Troubleshooting

## "agent mode is not enabled" / agentMode: false

**Symptom:** `skilltap status --json` returns `"agentMode": false`, or a command reports that agent mode is not enabled.

**Cause:** The `agent-mode.enabled` config flag is `false` (the default).

**Fix:** The user must run `skilltap config agent-mode` in an interactive terminal. This is TTY-only — the agent cannot run it.

Tell the user: "Please run `skilltap config agent-mode` in your terminal to enable agent mode, then retry."

## SECURITY ISSUE FOUND — INSTALLATION BLOCKED

**Symptom:** Exit 1 with a multi-line block starting with `SECURITY ISSUE FOUND — INSTALLATION BLOCKED`.

**Rules:**
1. Relay the entire block verbatim to the user. Do not summarize.
2. Do not retry the install.
3. Do not pass `--skip-scan` — it will be rejected if `require_scan = true` (the default).
4. Do not suggest disabling security settings.

The user must inspect the flagged file and lines manually and decide whether to install outside of agent-mode constraints via `skilltap install <url>` in their terminal.

## "Skill 'X' not found"

**Symptom:** `ERROR: Skill 'X' not found` or a skills info command returns nothing.

**Diagnosis steps:**
1. Run `skilltap skills --json` to list all installed skills and check spelling.
2. Check scope: `skilltap skills --project` and `skilltap skills --global` separately — the skill may be installed in the other scope.
3. Check if the skill is disabled: `skilltap skills --disabled`.

**Fix:** Install the skill with the correct scope, or pass `--project`/`--global` to the command to target the right scope.

## "Plugin 'X' is not installed"

**Symptom:** `plugin info` or `plugin toggle` reports the plugin is not installed.

**Diagnosis steps:**
1. Run `skilltap plugin --json` to list all installed plugins.
2. Check scope: `skilltap plugin --project` and `skilltap plugin --global` separately.

**Fix:** Install the plugin in the correct scope, or pass the right scope flag to the plugin command.

## Skill installed but agent can't find it

**Symptom:** A skill is listed by `skilltap skills` but the agent (e.g. Claude Code) does not load it.

**Cause:** The agent-specific symlink is missing. Skilltap installs to `.agents/skills/` canonically but only creates agent-dir symlinks if `--also <agent>` was passed at install time.

**Diagnosis:** Run `skilltap doctor` — it checks agent symlinks and reports broken or missing ones.

**Fix:** Re-run the install with `--also <agent>` to create the symlink:
```bash
skilltap install <source> --also claude-code
```
Or run `skilltap doctor --fix` to repair broken symlinks automatically.

## installed.json parse failure

**Symptom:** Any skilltap command fails with a JSON parse error mentioning `installed.json`.

**Diagnosis:** The JSON file is malformed (most commonly from a hand edit).

**Steps:**
1. Run `skilltap doctor` — it will report the exact parse error.
2. Advise the user to inspect and fix the file at the path reported by doctor (typically `~/.config/skilltap/installed.json` or `<project>/.agents/installed.json`).
3. After fixing, run `skilltap doctor` again to confirm.

## MCP server did not appear in agent

**Symptom:** A plugin was installed but its MCP server is not showing up in the agent's MCP list.

**Diagnosis:**
1. Run `skilltap plugin info <name> --json` and check the component's `active` field.
2. If `active: false`, the MCP component is toggled off.

**Fix:**
```bash
skilltap plugin toggle <name> --mcps
```

This flips the MCP components' active state. If they were `false`, they become `true` and are re-injected into the agent config file.

If the component is already `active: true` but the server still doesn't appear, the agent config file injection may have failed. Run `skilltap doctor` to check agent config file integrity.

## Modified agent config and want to recover pre-skilltap state

**Symptom:** After plugin install, `~/.claude/settings.json` (or equivalent) was modified and the user wants to restore the original.

**Recovery:** Before the first plugin install modified any config file, skilltap wrote a backup:

```
~/.claude/settings.json.skilltap.bak
```

The user can copy this backup back:
```bash
cp ~/.claude/settings.json.skilltap.bak ~/.claude/settings.json
```

Note: this restores the file to its pre-skilltap state. Any plugins that were working will stop working (MCP entries removed).

## --skip-scan rejected

**Symptom:** `ERROR: --skip-scan is not allowed when require_scan is enabled`.

**Cause:** `security.agent.require_scan = true` (the default) rejects `--skip-scan` in agent mode.

**Do not:** Try to pass `--skip-scan` again or advise the user to set it.

**Tell the user:** The security policy requires all installs to be scanned. If they want to install this skill without a scan, they must:
1. Run `skilltap config security` in their terminal to change the agent security policy.
2. Or install manually in their terminal (human mode allows more flexibility).

## skill remove fails on plugin-owned skill

**Symptom:** `skilltap skills remove <name>` reports the skill is not found or does nothing, even though the skill directory exists.

**Cause:** The skill is owned by a plugin and is tracked in `plugins.json`, not `installed.json`. `skills remove` only operates on records in `installed.json`.

**Fix:** Use plugin commands instead:
- To remove the entire plugin: `skilltap plugin remove <plugin-name>`
- To disable just the skill component: `skilltap plugin toggle <plugin-name> --skills` (check `plugin info` first to verify current state)

## Doctor categories

`skilltap doctor` checks:
- **git** — git is available on PATH
- **config** — `config.toml` loads and parses without error
- **dirs** — required directories exist and are accessible
- **installed.json** — parses without error; skill directories exist
- **skill integrity** — each installed skill has a valid SKILL.md
- **agent symlinks** — per-agent symlinks point to existing skill dirs
- **taps** — configured taps are reachable and valid
- **agent CLIs** — `security.agent_cli` path (if set) is executable
- **npm** — npm is available (if npm-sourced skills are installed)

`skilltap doctor --fix` repairs:
- Broken symlinks (removes and recreates)
- Orphaned records (removes `installed.json` entries with missing skill directories)

It does not make destructive changes to actual skill data.
