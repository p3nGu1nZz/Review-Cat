# AgentBusSystem (Swarm Control Plane)

## Overview

`AgentBusSystem` defines the **real-time control plane** for the dev harness swarm.

GitHub remains the **durable coordination layer** (Issues/PRs/comments/labels). The agent bus exists for:

- low-latency worker telemetry (liveness, stage, progress)
- structured error reporting (retryability, recovery hints)
- project/swarm snapshots (UI/visualizer)
- memory synchronization messages (catalog snapshots, engram announcements)
- compatibility enforcement (protocol drift + sync requirements)

MVP topology is **hub-and-spoke**:

- **Director broker** listens on a socket
- **workers** connect to the Director
- Director may optionally broadcast to operator clients (e.g., swarm visualizer)

All networking + timing knobs live in `config/dev.toml`.

## Requirements

### 1) Transport + framing

- Transport MUST be reliable and ordered (MVP: TCP).
- Messages MUST be framed so multiple JSON objects can be sent over one connection.

**Recommended MVP framing:** newline-delimited JSON (NDJSON).

- Each line is one JSON object.
- Lines MUST be \n-terminated.
- Payload size MUST be bounded (see Limits).

> Alternative (future): length-prefixed binary framing. Keep NDJSON for MVP simplicity.

### 2) Protocol envelope (normative)

Every agent-bus message MUST be wrapped in an envelope.

Envelope fields (required unless noted):

- `schema_version` (string) — envelope schema version (e.g., `"agentbus-envelope/v1"`)
- `message_type` (string) — stable message discriminator (see Message types)
- `sent_at` (string) — ISO-8601 UTC timestamp
- `sender` (object)
	- `role` (string) — `director | worker | operator`
	- `id` (string) — stable sender id (e.g., `worker-3-1707600000`)
	- `run_id` (string, optional) — director run/session id
- `seq` (integer) — monotonic sequence number per-connection
- `payload` (object) — message-specific body

Example (non-normative):

```json
{
	"schema_version": "agentbus-envelope/v1",
	"message_type": "heartbeat.v1",
	"sent_at": "2026-02-12T00:00:00Z",
	"sender": {"role": "worker", "id": "worker-3-1707600000", "run_id": "run_20260212-000000Z"},
	"seq": 42,
	"payload": {"state": {"stage": "CODE", "issue_number": 21}}
}
```

### 3) Version negotiation

- On connect, a worker MUST send `protocol_hello.v1`.
- The Director MUST respond with `protocol_welcome.v1` or `protocol_incompatibility.v1`.

Versioning rules:

- `protocol_version` is a semantic-ish string (e.g., `"0.1"`) owned by the repo.
- Minor version bumps MAY add optional fields.
- Breaking changes MUST bump major version and require explicit incompat handling.

### 4) Message categories (required)

#### A) Core telemetry/control

- `protocol_hello.v1`
- `protocol_welcome.v1`
- `heartbeat.v1`
- `task_progress.v1` (optional; may be folded into heartbeat)
- `worker_error.v1`
- `project_state_snapshot.v1`

#### B) Memory messages

- `memory_event.v1` (normalized wrapper; Director internal ingest)
- `engram_announce.v1`
- `engram_catalog_snapshot.v1`
- `engram_catalog_delta.v1` (optional MVP+)

#### C) Compatibility / drift prevention

- `sync_required.v1`
- `protocol_incompatibility.v1`

### 5) Limits (must-have)

- Director MUST enforce a maximum message size (bytes) and drop/flag oversized payloads.
- Workers MUST throttle heartbeat frequency and batch non-critical telemetry.

Recommended knobs (see `config/dev.toml`):

- `agent_bus.max_message_bytes`
- `agent_bus.heartbeat_interval_seconds`

### 6) Security posture (MVP)

- MVP assumes a **trusted Docker network**.
- `security = docker_network_only` means:
	- bind/listen only on Docker network interfaces
	- no auth at transport layer
	- do not carry secrets in payloads

Before multi-host/LAN mode:

- require `security = mTLS`
- define identity/cert issuance and rotation

### 7) Reconnect behavior

- Workers MUST reconnect on disconnect with bounded exponential backoff + jitter.
- Workers MUST re-send `protocol_hello.v1` after reconnect.
- Director SHOULD treat reconnect as a continuation if `sender.id` matches.

## DTOs (planning-level, normative fields)

### `protocol_hello.v1`

Sent by a worker/operator immediately after connect.

Payload:

- `protocol_version` (string)
- `capabilities` (array of string)
- `worker` (object)
	- `worker_id` (string)
	- `container_id` (string, optional)
	- `image_tag` (string, optional)
	- `worktree_path` (string, optional)
	- `branch_name` (string, optional)
- `repo` (object)
	- `main_sha_seen` (string, optional)
	- `main_ancestor_sha` (string, optional) — merge-base of worker branch and `main`
	- `catalog_hash_seen` (string, optional) — hash of `memory/catalog.json` as seen by sender

### `protocol_welcome.v1`

Sent by the Director after validating `protocol_hello.v1`.

Payload:

- `protocol_version` (string) — selected/confirmed
- `director` (object)
	- `run_id` (string)
	- `main_sha_expected` (string, optional)
	- `catalog_hash_expected` (string, optional)

### `heartbeat.v1` (WorkerState)

Workers send periodic heartbeats. The Director uses this for TTL/watchdog detection.

Payload:

- `state` (object)
	- `stage` (string) — `START | UPGRADE_CHECKPOINT | SYNC_MAIN | CONTEXT_LOAD | CODE | VALIDATE | COMMIT | PR_CREATE | REVIEW_REQUEST | DONE | RETRY_WAIT | BLOCKED`
	- `issue_number` (integer)
	- `attempt` (integer)
	- `progress` (object, optional)
		- `step` (string)
		- `percent` (number)
	- `repo` (object, optional)
		- `branch_name` (string)
		- `base_branch` (string) — usually `main`
		- `main_ancestor_sha` (string)
		- `catalog_hash_seen` (string)
	- `last_error_id` (string, optional)
	- `last_update_at` (string) — ISO-8601 UTC (worker-local)

### `worker_error.v1` (Structured error report)

Payload:

- `error` (object)
	- `error_id` (string) — stable id for dedupe
	- `worker_id` (string)
	- `issue_number` (integer)
	- `stage` (string)
	- `attempt` (integer)
	- `error_type` (string)
	- `is_transient` (boolean)
	- `should_retry` (boolean)
	- `escalate_to_human` (boolean)
	- `message` (string)
	- `exit_code` (integer, optional)
	- `container_id` (string, optional)
	- `worktree_path` (string, optional)
	- `last_stdout_tail` (string, optional)
	- `last_stderr_tail` (string, optional)

> The canonical retry/backoff + escalation rules live in `docs/dev/ERROR_HANDLING.md`.

### `project_state_snapshot.v1`

Director-emitted snapshot used by operator UI clients.

Payload:

- `release` (object, optional)
	- `release_id` (string)
	- `release_branch` (string)
	- `release_pr_number` (integer, optional)
- `workers` (array)
	- each is a recent `WorkerState` + liveness metadata (last heartbeat age)
- `queue` (object)
	- `eligible_issue_numbers` (array of integer)
	- `claimed_issue_numbers` (array of integer)

### `sync_required.v1`

Director → worker message requiring the worker to sync its branch with `main`.

Payload:

- `reason` (string) — e.g., `"behind_main" | "catalog_mismatch" | "protocol_change"`
- `main_sha_expected` (string)
- `catalog_hash_expected` (string, optional)
- `policy` (object)
	- `max_commits_behind` (integer, optional)
	- `max_seconds_behind` (integer, optional)
	- `require_catalog_hash_match` (boolean, optional)
- `required_action` (string) — `"merge_main" | "rebase_main"`

### `protocol_incompatibility.v1`

Director → worker message indicating a **hard incompatibility**.

Payload:

- `reason` (string)
- `expected_protocol_version` (string)
- `sender_protocol_version` (string)
- `required_action` (string) — typically `"sync_main_and_restart"`

### `engram_announce.v1`

Director announces a new engram exists.

Payload:

- `batch_id` (string) — UTC `YYYYMMDD-HHMMSSZ`
- `engram_id` (string)
- `scope` (string) — `st | lt`
- `path` (string) — `memory/{st|lt}/<batch_id>/engram_<engram_id>.json`
- `hash` (string)

### `engram_catalog_snapshot.v1`

Payload:

- `catalog_version` (string | integer)
- `catalog_hash` (string)
- `catalog_path` (string) — should be `memory/catalog.json`
- `entries` (array) — see `docs/specs/components/EngramCatalogDTO.md`

### `memory_event.v1`

A normalized wrapper used as input to the Director’s bounded event buffer.

Payload:

- `event_type` (string) — one of the bus `message_type` values
- `event` (object) — the original envelope (or a normalized subset)

## Configuration

`config/dev.toml` is canonical for:

- `[agent_bus]` networking + security
- drift policy knobs (see Worker sync policy)

Recommended keys (additive):

```toml
[agent_bus]
max_message_bytes = 65536
heartbeat_interval_seconds = 15

[policy.sync_main]
# If behind more than this threshold, Director sends sync_required.
max_commits_behind = 20
max_seconds_behind = 86400

# If true, catalog hash mismatch triggers sync_required.
require_catalog_hash_match = true
```

## Acceptance criteria

- Message envelope is defined and stable.
- Required message types are enumerated with normative fields.
- Heartbeat TTL + watchdog policy can be enforced by the Director using message data.
- Worker drift can be detected using `main_ancestor_sha` and `catalog_hash_seen`.

## References

- Issue #15 (Agent Bus)
- Issue #21 (Worker sync policy)
- Issue #9 (Error handling)
- Issue #13 (`MemorySyncSystem`)
- `docs/specs/systems/OrchestrationFSMSystem.md`
- `docs/specs/systems/RequestBudgetSystem.md`
