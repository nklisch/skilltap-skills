# Configuration

## File location

```
~/.config/skilltap/config.toml
```

If `$XDG_CONFIG_HOME` is set, it is used instead of `~/.config`. Config is **global only** — there is no per-project config file. Project-level dependencies live in `skilltap.toml` (see [manifest.md](manifest.md)), which is a different concept from `config.toml`.

## Full schema

```toml
# Top-level keys
verbose          = true                       # boolean — show per-step install details
default_git_host = "https://github.com"       # base URL for owner/repo shorthands
builtin_tap      = true                       # boolean — enable the built-in skilltap-skills tap

[defaults]
also  = []                  # string[] — agent symlink targets applied to every install
yes   = false               # boolean — auto-accept "do it?" prompts globally
scope = ""                  # "" | "global" | "project" — empty = smart-scope inference

[security]
scan    = "static"          # "semantic" | "static" | "none". Default: "static".
on_warn = "install"         # "prompt" | "fail" | "install". Default: "install".
trust   = []                # glob patterns matched against tap name OR source URL.
                            # Matches skip the scan entirely.

[scanner]
agent_cli    = ""           # path or name of agent CLI for semantic scans.
                            # "" prompts on first use, then persists the choice.
ollama_model = ""           # model name when agent_cli = "ollama"
threshold    = 5            # int 0..10 — semantic-chunk score gating
max_size     = 51200        # int — max skill dir size in bytes (default 50 KB)

[updates]
auto_update                = "off"   # "off" | "patch" | "minor"
interval_hours             = 24      # int — how often to check for skilltap updates
skill_check_interval_hours = 24      # int — how often to check for skill updates
show_diff                  = "full"  # "full" | "stat" | "none"

[telemetry]
enabled      = false        # do not set manually; use 'skilltap config telemetry'
notice_shown = false        # internal — set after first-run prompt
anonymous_id = ""           # internal — random UUID assigned on enable

[registry]
enabled = ["skills.sh"]     # registry names to search; [] disables all
sources = []                # custom RegistrySource entries

[[taps]]
name = "home"
url  = "https://gitea.example.com/nathan/my-tap"
```

**Enum values** (single source of truth in `core/src/schemas/config.ts`):

| Enum | Values | Default |
|---|---|---|
| `security.scan` | `semantic`, `static`, `none` | `static` |
| `security.on_warn` | `prompt`, `fail`, `install` | `install` |
| `defaults.scope` | `""`, `global`, `project` | `""` (smart) |
| `updates.auto_update` | `off`, `patch`, `minor` | `off` |
| `updates.show_diff` | `full`, `stat`, `none` | `full` |

## `[security]` vs `[scanner]`

- `[security]` is **policy** — *what should happen* (scan layer, what to do on warnings, which sources to trust unconditionally).
- `[scanner]` is **operational** — *how to run the semantic scan* (which agent CLI, threshold, max size).

There is no `[security.human]` / `[security.agent]` per-mode split, no `[[security.overrides]]` table-array, no `preset = ...` resolver, and no `require_scan` key. `--strict` on the CLI is equivalent to `on_warn = "fail"` for that one invocation. Legacy configs containing those shapes hard-fail at load — run `skilltap migrate` to translate.

## Schema enforcement

`loadConfig()` rejects legacy keys with an explicit error pointing at `skilltap migrate`. The following keys are not silently translated:

- `[security.human]`, `[security.agent]` (per-mode blocks)
- `[[security.overrides]]` (override array, including `preset = `)
- `require_scan = ` anywhere
- `[agent-mode]`, `[agent]`
- `[registry] allow_npm`

Run `skilltap migrate` once on each machine to convert.

## `config get` / `config set` / `config edit`

```bash
skilltap config get                      # print the entire config (pretty)
skilltap config get <key>                # read one key
skilltap config get <key> --json         # read one key as JSON

skilltap config set <key> <value>        # write one key (allowlisted keys only)
skilltap config edit                     # open ~/.config/skilltap/config.toml in $EDITOR
```

Examples:
```bash
skilltap config get security.on_warn
skilltap config set security.on_warn fail
skilltap config set defaults.scope project
skilltap config set defaults.also claude-code   # comma-separated → string[]
```

## Settable keys (allowlist)

`skilltap config set` accepts only the keys defined here. Source of truth: `core/src/config-keys.ts` (`SETTABLE_KEYS`).

| Key | Type |
|---|---|
| `defaults.scope` | `""` \| `"global"` \| `"project"` |
| `defaults.also` | comma-separated string → `string[]` |
| `defaults.yes` | `true` \| `false` |
| `scanner.agent_cli` | file path |
| `scanner.ollama_model` | string |
| `scanner.threshold` | integer 0..10 |
| `scanner.max_size` | integer (bytes) |
| `updates.auto_update` | `"off"` \| `"patch"` \| `"minor"` |
| `updates.interval_hours` | integer |
| `updates.skill_check_interval_hours` | integer |
| `updates.show_diff` | `"full"` \| `"stat"` \| `"none"` |
| `default_git_host` | URL string |
| `builtin_tap` | `true` \| `false` |
| `verbose` | `true` \| `false` |

## Blocked keys (use dedicated subcommands)

| Key(s) | Use this instead |
|---|---|
| `security.scan`, `security.on_warn`, `security.trust` | `skilltap config security` (or hand-edit `config.toml`) |
| `telemetry.enabled` | `skilltap config telemetry enable` / `disable` |
| `telemetry.notice_shown`, `telemetry.anonymous_id` | internal — do not set |
| `taps` | `skilltap tap add` / `tap remove` |

## `config security`

```bash
skilltap config security                            # interactive wizard (TTY)
skilltap config security --scan static              # set [security].scan
skilltap config security --on-warn fail             # set [security].on_warn
skilltap config security --trust-add "github.com/my-corp/*"   # append a trust glob
```

Non-interactive flags let you script the security policy without `config set`.

## `config telemetry`

```bash
skilltap config telemetry           # show current status
skilltap config telemetry enable    # generate anonymous_id, enabled = true
skilltap config telemetry disable   # enabled = false
```

Telemetry is anonymous and opt-in. Default off. Collects OS, architecture, CLI version, command name, success/failure, error type, installed-skill count, and command duration. Never collects skill names, repo URLs, paths, or PII.

`DO_NOT_TRACK=1` and `SKILLTAP_TELEMETRY_DISABLED=1` are honored as env-var kill-switches and also silence the first-run consent prompt.

## `config edit`

Opens `~/.config/skilltap/config.toml` in `$EDITOR`. After saving, no reload step is needed — the next skilltap command reads the updated file.
