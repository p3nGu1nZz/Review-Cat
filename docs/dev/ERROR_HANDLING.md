# Dev Harness Error Handling (Retries + Recovery)

This document defines how the **ReviewCat dev harness** (Director + workers) classifies failures, retries safely, and escalates when human input is required.

This is **planning-level** guidance; implementation lives in `dev/harness/*` once bootstrapped.

## Goals

- Avoid tight retry loops and cascading failures.
- Make transient failures self-healing where safe.
- Make non-retryable failures fail fast (so the swarm doesn’t churn).
- Ensure every failure is **observable** (agent bus + audit bundle + GitHub comment/label when appropriate).

## Error categories (normative)

### Transient (auto-retry)

These SHOULD be retried (bounded) with backoff + jitter:

- GitHub API/MCP timeouts
- GitHub rate limiting / abuse detection signals (cooldown)
- temporary network issues
- agent bus disconnects
- Docker daemon transient failures (pull hiccups, temporary resource pressure)

### Permanent (fail fast)

These SHOULD NOT be retried beyond a small sanity retry (often 0):

- deterministic compile errors introduced by the change
- deterministic test failures introduced by the change
- invalid config schema / missing required config
- missing required tools/prereqs (after a single re-check)

### Human-required (escalate)

These require human steering and MUST result in `agent-blocked`:

- contradictory or ambiguous requirements/specs
- repeated transient failures beyond retry budget (systemic outage)
- suspected policy violation (secrets, unsafe operations)

## Structured error reporting (agent bus DTO)

Workers report errors to the Director using `worker_error.v1` (see `docs/specs/systems/AgentBusSystem.md`).

The error payload MUST include enough information for the Director to decide:

- retry vs fail fast
- reclaim the issue vs keep it claimed
- kill/restart the container

### Error DTO (normative fields)

```json
{
	"error_id": "20260212-004102Z-worker-3-attempt-2",
	"worker_id": "worker-3-1707600000",
	"issue_number": 21,
	"stage": "VALIDATE",
	"attempt": 2,
	"error_type": "tests_failed",
	"is_transient": false,
	"should_retry": false,
	"escalate_to_human": true,
	"message": "Unit tests failed deterministically",
	"exit_code": 1,
	"container_id": "<docker-id>",
	"worktree_path": "../Review-Cat-agent-21-1707600000",
	"last_stdout_tail": "...",
	"last_stderr_tail": "..."
}
```

### Suggested `error_type` values

- `github_rate_limited`
- `github_mcp_timeout`
- `agent_bus_disconnect`
- `docker_exit_nonzero`
- `build_failed`
- `tests_failed`
- `invalid_config`
- `missing_prereq`

## Retry + backoff policy

Retry policy is **bounded**.

- Global max retries per task: `director.max_retries` in `config/dev.toml`.
- Backoff policy: `[retry]` in `config/dev.toml`.

Recommended behavior:

- Apply exponential backoff with jitter for transient categories.
- Honor `Retry-After` when provided by upstream (GitHub).
- Use circuit-breaker style cooldown for systemic problems (rate limits, widespread timeouts).

See also `docs/specs/systems/RequestBudgetSystem.md`.

## Director recovery rules (planning-level)

### Stale heartbeat / worker disconnect

If a worker heartbeat is stale beyond `timeouts.worker_heartbeat_ttl_seconds`:

1. Attempt to reconnect / re-establish telemetry.
2. If no progress, terminate the worker container.
3. Retry the task in a fresh worker (bounded by retry budget).
4. If repeated: reclaim the issue and apply `agent-blocked` with context.

### Container exit (non-zero)

- Collect exit code + logs tail.
- Classify:
	- transient: OOM kill, Docker daemon hiccup (retry)
	- permanent: build/test failures due to changes (fail fast)

### GitHub/MCP outage

- Enter cooldown (do not dispatch new tasks).
- Keep already-running workers going if safe (avoid losing in-flight edits).
- Resume when health returns (bounded by cooldown + budgets).

## Surfacing failures to GitHub

When escalation is required:

- apply `agent-blocked`
- post a structured comment summarizing:
	- what failed (stage + error_type)
	- last known attempt count
	- remediation suggestions (human action)

When a retry is happening, prefer a lightweight progress comment rather than spam.

## References

- Issue #9
- `docs/specs/systems/AgentBusSystem.md`
- `docs/specs/systems/OrchestrationFSMSystem.md`
- `docs/specs/systems/RequestBudgetSystem.md`
- `config/dev.toml`
