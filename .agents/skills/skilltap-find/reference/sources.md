# Source Formats for `skilltap install`

Every accepted form for the `source` argument. Always prefer passing `installRef` from `skilltap find --json` output rather than constructing a source string yourself.

## All accepted forms

| Form | Example | Notes |
|------|---------|-------|
| Tap lookup | `commit-helper` | Bare name; must not contain `/`; searched across all configured taps |
| Tap lookup pinned | `commit-helper@v1.2.0` | Specific git ref |
| Tap-defined plugin | `tap-name/plugin-name` | First segment must match a known tap name exactly |
| GitHub shorthand | `user/repo` | Resolves via `default_git_host` config (default: `https://github.com`) |
| GitHub explicit | `github:user/repo` | Always uses GitHub regardless of `default_git_host` |
| Git HTTP/HTTPS | `https://gitea.example.com/u/r` | Any git host |
| Git SSH | `git@github.com:user/repo.git` | Uses the user's local git auth |
| npm package | `npm:@scope/skill-name` | npm registry |
| npm pinned | `npm:@scope/skill-name@1.2.3` | Specific npm version |
| HTTP tarball | `https://…/skill.tar.gz` | Via HTTP registry source (`source.type: "url"`) |
| Local path | `./relative` or `/abs/path` or `~/path` | Links by default — not cloned |

## Which form to use

| Situation | Use |
|-----------|-----|
| User has a tap and the skill appears in `find` output | Bare name (tap lookup) or pass `installRef` directly |
| No taps, or skill not in any tap | Full URL or GitHub shorthand |
| Installing from a private self-hosted server | HTTPS or SSH URL |
| Installing a versioned npm skill | `npm:@scope/name@version` |
| Installing from a local directory (development) | `./path` or `/abs/path`; use `skilltap skills adopt` to copy instead of link |
| Installing a plugin bundled in a tap | `tap-name/plugin-name` |

## Resolution order

When `skilltap install` receives a source string, it resolves in this priority order:

1. **Tap name** — if the string has no `/`, no URL prefix, and no local-path prefix, it is tried as a tap skill name first. Fails fast if no taps are configured.
2. **Tap-defined plugin** — if the string is `x/y` (exactly one `/`, no URL prefixes), the first segment is matched against known tap names.
3. **Git URL** — strings starting with `https://`, `http://`, `git@`, or `ssh://`.
4. **npm** — strings starting with `npm:`.
5. **HTTP tarball** — HTTP URLs not matched by the git adapter (e.g. `.tar.gz` downloads).
6. **Local path** — strings starting with `./`, `/`, or `~/`.
7. **GitHub shorthand** — remaining `owner/repo` or `github:owner/repo` strings.

The tap name and tap-plugin checks run before `resolveSource` is called. If a bare name fails tap lookup, the error is returned immediately — it does NOT fall through to the GitHub shorthand adapter.

## Per-form gotchas

- **Tap lookup**: A bare name with no `/` is always treated as a tap lookup. If no taps are configured, you get an error before any network call.
- **Tap-defined plugin** (`tap-name/plugin-name`): The first segment must exactly match a tap name in your config. A `user/repo` form that does NOT match any tap name is passed to the GitHub shorthand adapter instead.
- **GitHub shorthand** (`user/repo`): Respects `default_git_host`. If your org uses Gitea, set `default_git_host = "https://gitea.example.com"` in config; then `user/repo` resolves against it. Use `github:user/repo` to force GitHub regardless.
- **Local paths** (`./`, `/`, `~/`): Installed by symlink, not cloned. Changes to the source directory are immediately reflected. To snapshot a copy instead, use `skilltap skills adopt`.
- **npm pinned versions**: The `@version` suffix goes after the package name: `npm:@scope/pkg@1.2.3` — not after `npm:`.
