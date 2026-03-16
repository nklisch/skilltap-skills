# Security configuration

Security settings are configured per-mode (human vs agent) with presets and trust overrides.

## Contents
- Configure security
- Presets
- Per-mode settings
- Trust overrides

## Configure security

```bash
skilltap config security [--preset P] [--mode M] [--scan S] [--on-warn W] [--require-scan] [--trust T] [--remove-trust R]
```

Flags:
- `--preset none|relaxed|standard|strict` — apply a preset
- `--mode human|agent|both` — target mode (default: both)
- `--scan static|semantic|off` — scan level
- `--on-warn prompt|fail|allow` — warning behavior
- `--require-scan` — block `--skip-scan` flag
- `--trust tap:name=preset|source:type=preset` — add trust override
- `--remove-trust name` — remove trust override

## Presets

| Preset | Scan | On warn | Require scan |
|--------|------|---------|--------------|
| none | off | allow | false |
| relaxed | static | allow | false |
| standard | static | prompt | false |
| strict | semantic | fail | true |

## Per-mode settings

Each mode (human, agent) has independent settings in `config.toml`:

```toml
[security.human]
scan = "static"
on_warn = "prompt"
require_scan = false

[security.agent]
scan = "static"
on_warn = "fail"
require_scan = true
```

Agent mode defaults are stricter than human mode.

## Trust overrides

Override security policy for specific taps or source types.

```bash
skilltap config security --trust tap:home=relaxed        # trust a specific tap
skilltap config security --trust source:npm=strict       # strict for all npm
skilltap config security --remove-trust home             # remove override
```

Matching priority: named tap match (exact) > source type match.

Source types: `tap`, `git`, `npm`, `local`
