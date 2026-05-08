# Agent Mode

## Mandatory status check

Before running any skilltap command, check agent mode is enabled:

```bash
skilltap status --json
```

Parse the JSON output. If `agentMode !== true`, stop and tell the user:

> Agent mode is not enabled. Please run `skilltap config agent-mode` in your terminal to enable it, then retry.

`skilltap config agent-mode` is TTY-only (interactive). The agent cannot run it — the user must do it.

## Status JSON schema

```json
{
  "agentMode": true,
  "scope": "project",
  "security": {
    "human": { "scan": "static", "on_warn": "prompt", "require_scan": false },
    "agent": { "scan": "static", "on_warn": "fail",   "require_scan": true  },
    "agent_cli": null
  },
  "also": ["claude-code"],
  "taps": 1,
  "plugins": 0
}
```

Fields:
| Field | Type | Meaning |
|---|---|---|
| `agentMode` | boolean | Must be `true` to proceed |
| `scope` | `"project"` \| `"global"` \| `"(not configured)"` | Default install scope |
| `security.agent.scan` | `"static"` \| `"semantic"` \| `"off"` | Active scan mode for agent installs |
| `security.agent.on_warn` | `"fail"` \| `"allow"` \| `"prompt"` | What happens on security warnings |
| `security.agent.require_scan` | boolean | Whether `--skip-scan` is rejected |
| `security.agent_cli` | string \| null | Path to agent CLI for semantic scans |
| `also` | string[] | Default `--also` targets from config |
| `taps` | number | Count of configured taps (including built-in if enabled) |
| `plugins` | number | Count of global plugins |

Plain-text format (one `key: value` per line, no `--json`):
```
agent-mode: enabled
scope: project
security.human: <human-readable description>
security.agent: <human-readable description>
agent_cli: <path>|(none)
also: claude-code|(none)
taps: 1
plugins: 0
```

## Agent mode detection

Agent mode is a config flag — NOT an environment variable. It is set by running `skilltap config agent-mode` interactively. There is no env var shortcut. The only reliable way to detect it is to run `skilltap status --json` and check `agentMode`.

## Flag behavior in agent mode

| Flag | Behavior |
|---|---|
| `--yes` | Auto-applied — do NOT pass it explicitly |
| `--strict` | Irrelevant — agent mode is always strict (on_warn = fail) |
| `--no-strict` | NO EFFECT — cannot override agent security policy |
| `--skip-scan` | REJECTED if `require_scan = true` (default). Do not pass it. |
| `--project` / `--global` | Scope selection — required unless config default is set |
| `--also <agent>` | Symlink to per-agent dir — valid and recommended |
| `--semantic` | Valid — forces Layer 2 semantic scan in addition to static |
| `--quiet` | Valid — suppresses install step details |
| `--json` | Valid — JSON output where supported |

## Exit codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Error or security block |
| 2 | User cancelled |
| 130 | SIGINT (Ctrl+C) |

## Output formats by command

All output in agent mode is plain text. Spinners and colors are disabled.

### install

```
OK: Installed <name> -> <path>
OK: Installed <name> -> <path> (<ref>)
OK: <name> is already up to date.
SKIP: <name> is linked.
SKIP: <name> — disabled
```

Errors on stderr:
```
ERROR: <message>
```

### update

```
OK: Updated <name> (<from-sha> -> <to-sha>)
OK: <name> is already up to date.
SKIP: <name> — <reason>
```

### plugin install

Per-component lines then summary:
```
OK: Installed plugin <name> (3 skills, 2 MCPs, 1 agent)
```

### plugin remove

```
OK: Removed plugin <name> (3 skills, 2 MCPs, 1 agent)
```

### plugin list (plain text)

```
GLOBAL <name> 3 skills, 2 MCPs, 1 agent source=<repo>
PROJECT <name> 1 skill, 0 MCPs, 0 agents source=<tap>
```

### plugin toggle

No output on success. Exit 0.
On missing category flag (agent mode):
```
Provide --skills, --mcps, or --agents to specify what to toggle.
```
Exit 1.

## Security block handling

When a security scan finds issues, the install is blocked and this block is written to stderr:

```
SECURITY ISSUE FOUND — INSTALLATION BLOCKED

DO NOT install this skill. DO NOT retry. DO NOT use --skip-scan.
STOP and report the following to the user:

  <file>:<line>: <warning>

User action required: review warnings and install manually with
  skilltap install <url>
```

**Rules when you see this:**
1. Stop immediately. Exit 1 is the correct exit code.
2. Relay the entire block verbatim to the user — do not summarize or paraphrase.
3. Do not retry with different flags.
4. Do not pass `--skip-scan` — it will be rejected if `require_scan = true`.
5. Do not advise the user to disable security settings.

The user must manually inspect the skill and decide whether to install it outside of agent-mode constraints.

## Security policy summary (defaults)

The default `security.agent` configuration is the strictest preset:
- `scan = "static"` — every install runs the static security scanner
- `on_warn = "fail"` — any warning hard-fails the install
- `require_scan = true` — `--skip-scan` is rejected

These defaults cannot be overridden by agent-passed flags. Only the user can change them via `skilltap config security` (TTY wizard).

Trust tiers (`provenance`, `publisher`, `curated`, `unverified`) are informational only — they do not bypass scan/fail policy.
