# CopilotRunnerSystem

## Overview

`CopilotRunnerSystem` is responsible for invoking GitHub Copilot CLI and
capturing outputs in a reproducible way. It is the boundary between ReviewCat
logic and the external agent.

The system supports GitHub MCP Server configuration so that agents invoked
through it can access GitHub tools (issues, PRs, comments) natively.

## Requirements

1. Must run Copilot CLI in programmatic mode for persona prompts.
2. Must capture stdout/stderr and exit code.
3. Must write prompt ledger entries for each invocation.
4. Must support record/replay mode for tests.
5. Must enforce tool allow/deny policy.
6. Must support `--mcp-config` for GitHub MCP Server integration.
7. Must support agent profile references (`@dev/agents/<role>.md`).
8. Must be non-interactive: if Copilot CLI requests interactive input (trust
	confirmations, auth flows), the call must fail fast with a clear error so the
	Director can escalate via `agent-blocked`.

## Interfaces

Inputs:

- `PromptRecord`
- policy (allow/deny lists)
- MCP config path (optional, for GitHub MCP Server)
- agent profile path (optional, e.g., `@dev/agents/coder.md`)

Outputs:

- raw output text
- parsed JSON (best effort)
- ledger entry

Ledger files:

- `ledger/copilot_prompts.jsonl`
- `ledger/copilot_raw_outputs/<call_id>.txt`

## Acceptance criteria

- Given a prompt, system produces a raw output and a ledger record.
- In replay mode, system returns fixture output without calling Copilot CLI.
- Denied tools are never requested in programmatic options.

## Test cases

- Replay mode returns fixture.
- Ledger entry is written with correct timestamps.
- Policy merges config + CLI overrides.

## Edge cases

- Copilot CLI not installed.
- Not authenticated.
- Copilot CLI prompts for trust confirmation.
- GitHub MCP Server not configured or unavailable.

## Non-functional constraints

- Never print tokens.
- Timeouts for long-running calls.
- MCP config path must not contain secrets (env vars only).
- MCP Server is configured via either remote MCP or a native `github-mcp-server stdio` binary (host or container-bundled).
- All invocations are logged via LoggingSystem (spdlog in C++, log.sh in bash).
