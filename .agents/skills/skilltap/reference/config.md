# Configuration

## File location

```
~/.config/skilltap/config.toml
```

If `$XDG_CONFIG_HOME` is set, it is used instead of `~/.config`. Config is **global only** — there is no per-project config file.

## Full schema

```toml
# Default install behavior
[defaults]
also = []                    # string[] — agent symlink targets applied to every install
yes = false                  # boolean — auto-accept prompts globally
scope = ""                   # "" | "global" | "project" — default install scope

# Security — shared settings
[security]
agent_cli = ""               # path to agent CLI for semantic scans
threshold = 5                # int 0..10 — semantic risk score threshold
max_size = 51200             # int — max skill directory size in bytes (default 50 KB)
ollama_model = ""            # string — model name when using ollama agent for semantic scans

# Security — human-mode policy
[security.human]
scan = "static"              # "static" | "semantic" | "off"
on_warn = "prompt"           # "prompt" | "fail" | "allow"
require_scan = false         # boolean

# Security — agent-mode policy (defaults are intentionally stricter)
[security.agent]
scan = "static"              # "static" | "semantic" | "off"
on_warn = "fail"             # "prompt" | "fail" | "allow"
require_scan = true          # boolean — if true, --skip-scan is rejected

# Trust overrides per tap or source type
[[security.overrides]]
match = "home"               # tap name OR source type identifier
kind = "tap"                 # "tap" | "source"
preset = "relaxed"           # "none" | "relaxed" | "standard" | "strict"

# Agent mode settings (toggle via 'skilltap config agent-mode')
["agent-mode"]
enabled = false              # boolean — do not set manually; use the subcommand
scope = "project"            # "global" | "project" — default scope for agent installs

# Skill registry search
[registry]
enabled = ["skills.sh"]      # string[] — registry names to search; [] disables all
sources = []                 # custom search-API endpoint URLs

# User-added taps (each tap is an array element)
[[taps]]
name = "home"
url = "https://example.com/my-tap"
type = "git"                 # "git" | "http"
auth_token = ""              # optional bearer token (prefer auth_env)
auth_env = ""                # optional — name of env var holding bearer token

# Update behavior
[updates]
auto_update = "off"          # "off" | "patch" | "minor"
interval_hours = 24          # int — how often to check for updates
skill_check_interval_hours = 24
show_diff = "full"           # "full" | "stat" | "none"

# Telemetry (toggle via 'skilltap config telemetry')
[telemetry]
enabled = false              # boolean — do not set manually; use the subcommand
notice_shown = false         # internal
anonymous_id = ""            # internal

# Top-level keys
builtin_tap = true           # boolean — enable the built-in skilltap-skills tap
verbose = true               # boolean — show per-step install details
default_git_host = "https://github.com"  # base URL for user/repo shorthands
```

## config get / set

Read any config key:
```bash
skilltap config get defaults.scope
skilltap config get security.agent.on_warn
```

Write allowlisted keys:
```bash
skilltap config set defaults.scope project
skilltap config set security.threshold 7
```

## Settable keys (allowlist)

| Key | Type |
|---|---|
| `defaults.scope` | `""` \| `"global"` \| `"project"` |
| `defaults.also` | comma-separated string → string[] |
| `defaults.yes` | `true` \| `false` |
| `security.agent_cli` | file path |
| `security.ollama_model` | string |
| `security.threshold` | integer 0..10 |
| `security.max_size` | integer (bytes) |
| `updates.auto_update` | `"off"` \| `"patch"` \| `"minor"` |
| `updates.interval_hours` | integer |
| `updates.show_diff` | `"full"` \| `"stat"` \| `"none"` |
| `default_git_host` | URL string |

## Blocked keys (use dedicated subcommands)

| Key(s) | Redirect |
|---|---|
| `agent-mode.enabled`, `agent-mode.scope` | `skilltap config agent-mode` |
| `telemetry.enabled` | `skilltap config telemetry enable` / `disable` |
| `telemetry.notice_shown`, `telemetry.anonymous_id` | internal — do not set |
| `security.human.*`, `security.agent.*`, `security.overrides` | `skilltap config security` |
| `taps` | `skilltap tap add` / `tap remove` |

Attempting to `config set` a blocked key will print a redirect message and exit 1.

## Config subcommands

### agent-mode

```bash
skilltap config agent-mode
```

TTY-only interactive toggle. Cannot be run by the agent — user must run it in a terminal.

### security

```bash
skilltap config security
```

TTY wizard for configuring `security.human.*`, `security.agent.*`, and `security.overrides`. Can also accept flags for non-interactive use in human mode.

### telemetry

```bash
skilltap config telemetry         # show current status
skilltap config telemetry enable
skilltap config telemetry disable
```

### config get

```bash
skilltap config get              # print all config (pretty)
skilltap config get <key>        # print single key value
```

No allowlist restriction — any key can be read.

### config set

```bash
skilltap config set <key> <value>
```

Only allowlisted keys can be written. Values are type-coerced (e.g. `"true"` → boolean, `"5"` → integer).
