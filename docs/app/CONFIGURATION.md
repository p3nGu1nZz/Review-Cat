# Runtime app configuration (`reviewcat.toml`) (planned)

This repository is currently **planning/specs-first**. The runtime application (`reviewcat`) will eventually expose a user-facing configuration file.

## Goals

- Keep end-user configuration separate from dev-harness configuration.
- Make defaults safe and predictable.
- Keep secrets out of tracked config by default.

## Proposed file locations

- User-global: `~/.reviewcat/reviewcat.toml`
- Repo-local override: `./reviewcat.toml` (optional)

## Proposed precedence

1. CLI flags
2. Environment variables
3. Repo-local `reviewcat.toml`
4. User-global `~/.reviewcat/reviewcat.toml`
5. Built-in defaults

## Proposed sections (non-normative)

### `[review]`

- default personas enabled
- output formats (markdown/json)
- include/exclude paths

### `[personas]`

- persona prompt templates
- severity thresholds

### `[ui]`

- enable SDL3 UI
- theme colors / font scale

### `[logging]`

- log level
- log file path

## Related

- Dev harness config: `docs/dev/CONFIGURATION.md` and `config/dev.toml`
- `PLAN.md` (runtime architecture)
