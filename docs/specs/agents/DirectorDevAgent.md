# DirectorDev agent spec

## Overview

The DirectorDev agent coordinates development of the ReviewCat codebase by managing role agents (implementer, QA, docs, security, code-review).

This is intended to be implemented using Copilot CLI custom agents and a thin orchestration wrapper.

## Requirements

1. Must operate spec-first: implementation tasks must reference a spec file.
2. Must coordinate role agents in a predictable order.
3. Must run build/test after implementation.
4. Must record a development audit bundle.
5. Must operate in release cycles (batch issues into `feature/release-*` and finalize via merge agent).
6. Must use gitignored `STATE.json` for local cached state (first-run vs resume; active release context).
7. Must enforce the **issue-claim lock protocol** (Director-only claiming; label transitions + claim comments).

8. Must maintain a worker registry and enforce telemetry invariants:
  - workers must send periodic heartbeats (TTL enforced)
  - workers must report stage/progress and structured errors
  - stale/disconnected workers must be recovered (bounded retries) or escalated

9. Must enforce drift-prevention sync policy:
  - workers must ingest the latest `main` before PR readiness
  - memory catalog hash mismatches may trigger required sync
  - enforcement thresholds come from `config/dev.toml` (`[policy.sync_main]`)

Canonical label taxonomy and claim-comment format: `docs/dev/GITHUB_LABELS.md`.

## Interfaces

- Entry (recommended): `dev/scripts/daemon.sh` (keep-alive supervisor that starts `dev/harness/director.sh`)
- Entry (direct): `dev/harness/director.sh` (heartbeat daemon)
- Manual cycle: `dev/harness/run-cycle.sh <issue> <branch> <base_branch>`
- Artifact output: `dev/audits/<audit_id>/`

Role agents are defined under `.github/agents/` and `dev/agents/` as markdown
prompt files, invoked via `copilot -p @dev/agents/<role>.md "..."`.

## Acceptance criteria

- Given a spec, DirectorDev produces:
  - code changes implementing acceptance criteria
  - tests
  - updated docs
  - a prompt ledger

## Test cases

- Run DirectorDev in replay mode (no Copilot calls) using fixtures.

## Edge cases

- Spec incomplete or ambiguous.
- A role agent produces conflicting recommendations.
- Director/worker restart mid-release (must resume using `STATE.json` cache).

## Non-functional constraints

- Scope control: DirectorDev must refuse to expand scope beyond spec unless explicitly authorized.
- Safety: no network actions without opt-in.
- Logging: every heartbeat iteration logged via `dev/harness/log.sh`.
- MCP Server is configured via either remote MCP or a native `github-mcp-server stdio` binary (host or container-bundled).

## References

- `docs/specs/systems/ReleaseCycleSystem.md`
- `docs/specs/systems/OrchestrationFSMSystem.md`
- `docs/specs/systems/AgentBusSystem.md`
- `AGENT.md` (swarm contract)
