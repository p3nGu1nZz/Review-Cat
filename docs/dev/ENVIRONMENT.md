# Environment variables (Dev Harness + Tooling)

This repo is currently **planning/specs-first**. These environment variables define the intended configuration surface for the **dev harness** and its integrations.

Secrets must **never** be committed. Use `.env` (gitignored) locally.

## Files

- `.env.example` — committed template
- `.env` — local copy (gitignored)

## GitHub authentication

### `GITHUB_PERSONAL_ACCESS_TOKEN` (secret)

Used by `github-mcp-server` when running in **stdio** mode.

- Required for: `dev/mcp/github-mcp-stdio.json`
- Not required for: `dev/mcp/github-mcp.json` (remote HTTP MCP)

Never log this value.

### `GITHUB_TOOLSETS` (optional)

Optional toolset filter for `github-mcp-server`.

Example:

- `issues,pull_requests,repos,git`

## MCP selection

### `MCP_SERVER_URL` (non-secret)

URL for the **remote HTTP MCP** server (MVP default):

- `https://api.githubcopilot.com/mcp/`

If you use the remote MCP server, you typically run Copilot CLI with:

- `--mcp-config dev/mcp/github-mcp.json`

### `GITHUB_MCP_SERVER_PATH` (non-secret)

Path or command name for the local `github-mcp-server` binary.

In most setups this is simply:

- `github-mcp-server`

If you use the local stdio MCP server, run Copilot CLI with:

- `--mcp-config dev/mcp/github-mcp-stdio.json`

## Dev harness overrides (planning-level)

The canonical dev harness configuration is `config/dev.toml` (tracked).

For convenience (especially in CI, or when scripting), the dev harness may also support a small set of **environment overrides**.

These are **intended** (not yet implemented):

- `REVIEWCAT_DEV_CONFIG` — path to `config/dev.toml` (default: `config/dev.toml`)
- `REVIEWCAT_DIRECTOR_INTERVAL_SECONDS`
- `REVIEWCAT_MAX_WORKERS`
- `REVIEWCAT_DOCKER_IMAGE`
- `REVIEWCAT_DOCKER_WORKDIR`
- `REVIEWCAT_AGENT_BUS_LISTEN_ADDR`
- `REVIEWCAT_AGENT_BUS_LISTEN_PORT`

## Precedence rules

### Dev harness

1. CLI flags (when implemented)
2. Environment overrides (when implemented)
3. `config/dev.toml`
4. Built-in defaults

### Runtime app (planned)

1. CLI flags
2. Environment variables
3. `reviewcat.toml`
4. Built-in defaults

## Related docs

- `docs/dev/CONFIGURATION.md`
- `docs/app/CONFIGURATION.md`
- `PLAN.md` (MCP section)
