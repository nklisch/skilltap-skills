# Project Manifest and Lockfile

When a project has a `skilltap.toml` at the root, it becomes the source of truth for that project's installed skills, plugins, and standalone MCP servers. Together with `skilltap.lock`, the manifest is what gets committed to source control. `state.json` is local-only machine state — never committed.

## Files

| File | Path | Committed? | Purpose |
|---|---|---|---|
| Manifest | `<projectRoot>/skilltap.toml` | Yes | Declared dependencies + targets. |
| Lockfile | `<projectRoot>/skilltap.lock` | Yes | Auto-managed: exact resolved refs/SHAs. |
| State | `<projectRoot>/.agents/state.json` | No | Local on-disk state (per-machine). |

`<projectRoot>` is determined by walking up from CWD looking for `.git` or `skilltap.toml`. If neither is found, the cwd is used.

## Manifest schema (`skilltap.toml`)

```toml
# skilltap.toml — project root

[targets]
also  = ["claude-code", "cursor"]   # default agent symlinks for installs
scope = "project"                   # "project" | "global"

[skills]
"github:nathan/commit-helper" = "^1.0"
"npm:@corp/code-review"       = "*"
"local:./vendor/team-tools"   = "*"
"home/git-workflow"           = "*"   # tap-name/skill-name shorthand

[plugins]
"github:corp/dev-toolkit" = "*"
"home/team-bundle" = { ref = "v2.1", components = { "test-skipper" = false } }

[[mcps]]
name   = "search"
source = "github:corp/search-mcp"
ref    = "main"
also   = ["claude-code"]

[taps]
home = "https://gitea.example.com/nathan/my-tap"
```

### Tables

| Table | Shape | Notes |
|---|---|---|
| `[targets]` | `also: string[]`, `scope: "project" \| "global"` | Defaults applied to installs originating from this manifest. |
| `[skills]` | Map `<source>` → `<range>` or inline table | Declared skill dependencies. Source is `github:`, `npm:`, `local:`, `git:`, or `tap-name/skill-name`. Range is `"*"`, `"^1.0"`, `"v1.2.3"`. |
| `[plugins]` | Same shape as `[skills]` | Inline tables can disable specific components: `components = { "name" = false }`. |
| `[[mcps]]` | Array of inline tables | Standalone MCP server installs. First-class records keyed by user-chosen install name. The `ref` is an exact pin (no separate `range` field). |
| `[taps]` | Map `<name>` → `<git-url>` | Taps the project depends on. Synced into `[[taps]]` config on `sync --apply`. |

### Inline-table semantics

A skill or plugin entry written as `{ ref = "x" }` means **range = `"*"`**; the actual pin is the `sha` in the corresponding lockfile entry. `sync` does NOT report `ref-mismatch` for inline-table entries against a lockfile range of `*` — they're treated as "match whatever the lockfile says."

## Lockfile schema (`skilltap.lock`)

Auto-managed by every state-mutating command. Cargo-style:

```toml
# skilltap.lock — auto-managed
version = 1

[[skill]]
source = "github:nathan/commit-helper"
ref    = "v1.2.0"
sha    = "abc123def456..."
range  = "^1.0"

[[plugin]]
source = "github:corp/dev-toolkit"
ref    = "main"
sha    = "789abc..."
range  = "*"

[[mcps]]
name   = "search"
source = "github:corp/search-mcp"
ref    = "main"
sha    = "1a2b3c..."
also   = ["claude-code"]
```

Every record stores `source`, `ref` (resolved), `sha` (commit SHA), and `range` (the manifest's declared range — used by `update` to know what's allowed).

## Lifecycle drift

Every state-writing command keeps manifest + lockfile in lockstep when project scope is in play: `install`, `update`, `remove`, `move`, `adopt`, `toggle`, `migrate`. There is no "manifest gets out of date" path through the CLI; drift can only appear via manual edits to `skilltap.toml` / `skilltap.lock`, which `sync` reconciles.

## `sync` semantics

```bash
skilltap sync [--apply] [--strict] [--json]
```

Reconciles three sources of truth: `skilltap.toml`, `skilltap.lock`, `state.json`.

**Default behavior** (no flags): scan all three, print a drift report grouped by kind. If everything agrees, prints `✓ In sync. Manifest, lockfile, and state agree.` and exits 0. Otherwise prints drift items (target, source, reason, declared/installed/locked refs) and ends with `note: run skilltap sync --apply to execute this plan.`

**`--apply`** executes the plan via `install`/`remove`. Order: removals first, then ref-changes, then adds, then bookkeeping (lockfile-* categories).

**`--strict`** (only meaningful with `--apply`): stop on first failure.

**`--json`** outputs the plan as JSON instead of human-readable text.

**Project-root requirement.** `sync` resolves the project root via `findManifestRoot()` (walks up looking for `skilltap.toml`) with a fallback to `isInGitRepo()`. If neither exists, exits 1: `skilltap sync requires a project root (looks for .git or skilltap.toml).`

### Drift categories (`DriftKind`)

| Kind | Meaning | Action under `--apply` |
|---|---|---|
| `add` | Declared in manifest but not installed. | Install at locked ref (or resolve range if no lockfile entry yet). |
| `remove` | Installed but not declared. | Uninstall. |
| `ref-mismatch` | Declared with a different ref than locked (manifest is source of truth on conflict). | Update lockfile. |
| `lock-stale` | Locked SHA differs from installed SHA (lockfile is source of truth on disk). | Reinstall to match lockfile. |
| `lock-missing` | Installed but no lockfile entry. | Write lockfile entry from installed state. |
| `lock-orphan` | Lockfile entry with no manifest declaration. | Drop lockfile entry. |

`sync` reconciles all three state types (skills, plugins, mcps) in one pass.

## Publish manifest (`.skilltap/<name>.toml`)

A **separate** file from the project manifest. The project manifest declares what to install; the publish manifest declares what your repo publishes.

A repo opts into being a publishable plugin by adding one or more files under `.skilltap/<plugin-name>.toml`. The native publish format is **TOML**. `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` are read as inputs and normalized internally.

```toml
# .skilltap/team-toolkit.toml

name        = "team-toolkit"
version     = "1.0.0"
description = "Internal dev tools"
publish     = true   # required, default false; explicit opt-in

[[skills]]
name = "code-review"
path = "./skills/code-review"

[[skills]]
name = "lint-checker"
path = "./skills/lint-checker"

[[servers]]                       # MCP servers
name    = "db"
type    = "stdio"                 # "stdio" | "http"
command = "node"
args    = ["./mcp/db.js"]
env     = { DATABASE_URL = "${DATABASE_URL}" }

[[servers]]
name    = "search"
type    = "http"
url     = "https://search.internal.corp/mcp"
headers = { Authorization = "Bearer ${SEARCH_TOKEN}" }

[[agents]]
name = "reviewer"
path = "./agents/reviewer.md"
```

`publish = false` (or omitted) makes the manifest project-internal — the repo can still be installed for its consumer-side `[skills]` / `[plugins]` deps, but the plugin is not exposed to outside installers.

Multi-plugin: drop multiple files into `.skilltap/`. Each is independently publishable. Install one with `install plugin user/repo:plugin-name` or all with `:*`.

For full plugin manifest details (including `.claude-plugin/plugin.json` and `.codex-plugin/plugin.json` formats), see the `skilltap-author` skill.

## Common workflow

```bash
# Team lead: declare project deps
cd my-app
skilltap install skill commit-helper --scope project    # appends to skilltap.toml + lock
skilltap install plugin corp/dev-toolkit --scope project

# Commit both files
git add skilltap.toml skilltap.lock
git commit -m "Add skilltap dependencies"
git push

# Teammates: bring machine to parity
git pull
skilltap sync --apply
```

After `sync --apply`, `state.json` should have one record per declared item, matching the lockfile SHAs. Run `skilltap doctor` to verify.
