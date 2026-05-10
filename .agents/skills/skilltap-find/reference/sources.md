# Source Formats for `skilltap install`

Every accepted form for the `<source>` argument. Always prefer passing `installRef` from `skilltap find --json` output rather than constructing a source string yourself.

The same source forms work for `install skill`, `install plugin`, and `install mcp`. The type subcommand decides what skilltap looks for in the resolved source.

## All accepted forms

| Form | Example | Notes |
|---|---|---|
| Tap-resolved name | `commit-helper` | Bare name; must not contain `/`; searched across all configured taps. |
| Tap-resolved name + version | `commit-helper@v1.2.0` | Specific git ref. |
| Tap-defined plugin | `tap-name/plugin-name` | First segment must match a known tap name exactly. |
| GitHub shorthand | `user/repo` | Resolves via `default_git_host` config (default: `https://github.com`). |
| GitHub explicit | `github:user/repo` | Always uses GitHub regardless of `default_git_host`. |
| GitHub @ ref | `user/repo@v1.0` | Branch or tag suffix. |
| Plugin selector | `user/repo:plugin-name` | Plugin only — picks one plugin from a multi-plugin repo. |
| Plugin all | `user/repo:*` | Plugin only — installs every publishable plugin in the repo. |
| Plugin @ ref + selector | `user/repo@v1.0:frontend` | Compose. |
| Git HTTP/HTTPS | `https://gitea.example.com/u/r` | Any git host. |
| Git SSH | `git@github.com:user/repo.git` or `ssh://git@host/path` | Uses your local git auth. |
| npm package | `npm:@scope/name` | npm registry. Pin with `@version` suffix: `npm:@scope/name@1.2.3` or `npm:@scope/name@^1.0.0`. |
| Local path | `./relative` or `/abs/path` or `~/path` | Symlinked, not copied. |

## Which form to use

| Situation | Use |
|---|---|
| Skill appears in `find` output | Pass `installRef` directly. |
| No taps, or skill not in any tap | Full URL or GitHub shorthand. |
| Installing from a private self-hosted git server | HTTPS or SSH URL. |
| Installing a versioned npm skill | `npm:@scope/name@version`. |
| Installing from a local directory (development) | `./path` or `/abs/path`. Use `skilltap adopt` if you want a copy/move instead of a symlink. |
| Installing one plugin from a multi-plugin repo | `user/repo:plugin-name` (works with all source types — github shorthand, full URL, SSH). |
| Installing every plugin from a multi-plugin repo | `user/repo:*`. |

## Resolution order

When skilltap resolves a source string, adapters are tried in priority order:

1. **GitHub** (`github` adapter) — recognizes `user/repo`, `github:user/repo`, `https://github.com/...`, `git@github.com:user/repo`, plus `@ref` and `:plugin` suffixes.
2. **Git** — full https / ssh URLs to any git host. Same `@ref` and `:plugin` parsing.
3. **Local** — paths starting with `./`, `../`, `/`, or `~/`. Same `:plugin` suffix honored.
4. **npm** — strings starting with `npm:`.
5. **Tap** — any `tap-name/entry-name` where the first segment matches a configured tap name.

Tap-name lookup runs ahead of these adapters when the source is a bare name with no `/`, no URL prefix, and no local-path prefix. If no taps are configured and the source is a bare name, you get a fail-fast error before any network call.

`default_git_host` lets non-GitHub orgs use the bare `user/repo` shorthand against their own host (e.g. set it to `https://gitea.example.com` so `myteam/repo` resolves there).

## Per-form gotchas

- **Tap lookup**: A bare name with no `/` is always treated as a tap lookup. If no taps are configured, you get an error immediately — it does NOT fall through to the GitHub shorthand adapter.
- **Tap-defined plugin** (`tap-name/plugin-name`): the first segment must exactly match a tap name. A `user/repo` form that does NOT match any tap name is passed to the GitHub shorthand adapter instead.
- **GitHub shorthand** (`user/repo`): respects `default_git_host`. To force GitHub regardless of configuration, use `github:user/repo`.
- **Local paths**: installed by symlink, not cloned. Changes to the source directory are immediately reflected. To snapshot a copy instead, use `skilltap adopt <path> --move`.
- **npm pinned versions**: the `@version` suffix goes after the package name: `npm:@scope/pkg@1.2.3`, NOT after `npm:`.
- **Plugin selector parser**: `:plugin-name` is split off the **last** `:` after stripping `@ref`. HTTPS URLs (`https://...`) are unaffected. SSH URLs use the `:` after the host (`git@github.com:owner/repo`) — the parser knows to ignore that one and only treat a final `:plugin-name` as the selector.

## Type-aware install

After picking a source, choose the verb based on what the source contains:

| Source contents | Verb |
|---|---|
| `SKILL.md` (single or multi) | `install skill <source>` |
| `.skilltap/<name>.toml`, `.claude-plugin/plugin.json`, or `.codex-plugin/plugin.json` | `install plugin <source>` |
| Standalone MCP server (no SKILL.md, no plugin manifest) | `install mcp <source>` |

If the source doesn't match the chosen type, skilltap errors with a hint:

```
error: No SKILL.md found in owner/dev-toolkit.
hint: This source looks like a plugin. Try: skilltap install plugin owner/dev-toolkit
```

`skilltap try <type> <source>` previews the source (clones, parses, scans, prints structure) without installing — useful for vetting an unknown source before committing.
