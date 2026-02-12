# Dev Harness configuration (`config/dev.toml`)

`config/dev.toml` is the **canonical**, **tracked**, non-secret configuration for the ReviewCat dev harness.

It covers:

- Director scheduling and retry policy
- worker liveness timeouts
- Docker container execution (one image tag, many containers)
- agent bus networking (ports / message limits)
- memory sync budgets and paths
- request pacing for GitHub + model calls

Secrets (tokens) are **not** stored in TOML. See `docs/dev/ENVIRONMENT.md`.

## Configuration layers

For the dev harness, the intended precedence is:

1. CLI flags (planned)
2. Environment overrides (planned; see `docs/dev/ENVIRONMENT.md`)
3. `config/dev.toml`
4. Built-in defaults

Today, `config/dev.toml` is the authoritative planning artifact.

## Sections

### `[director]`

- `interval_seconds` — Director heartbeat loop interval
- `max_workers` — maximum concurrent worker containers
- `max_retries` — maximum end-to-end retries per task before escalation

### `[timeouts]`

- `worker_heartbeat_ttl_seconds` — time since last `heartbeat.v1` before a worker is considered disconnected
- `worker_watchdog_seconds` — hard timeout for a worker task cycle
- `issue_claim_stale_seconds` — how long a claimed issue can go without progress before reclaim

### `[retry]`

- `backoff_mode` — `exponential | linear`
- `backoff_initial_seconds`
- `backoff_max_seconds`

### `[containers]`

Defines the Docker execution model:

- `image` — **single shared image tag** used by Director and all workers
- `workdir` — bind-mount target inside containers (default: `/workspace`)
- `scale_to_zero_idle_seconds` — stop/remove idle worker containers

Related:

- Issue #2 (runtime model: containers + worktrees)
- Issue #14 (image lifecycle + tagging)

### `[agent_bus]`

Agent bus = swarm control plane (real-time telemetry + control). See `docs/specs/systems/AgentBusSystem.md`.

- `mode` — `director_broker` for MVP
- `listen_addr`, `listen_port` — Director broker bind address/port
- `security` — MVP: `docker_network_only` (trusted Docker network)
- `max_message_bytes` — hard cap for a single framed message
- `heartbeat_interval_seconds` — worker heartbeat frequency

### `[policy.sync_main]`

Drift-prevention policy: when workers must ingest `main` and/or memory updates.

- `max_commits_behind`
- `max_seconds_behind`
- `require_catalog_hash_match` — if true, catalog hash mismatch forces sync

### `[memory]`

Defines shared memory budgets/paths and the tracked focus view.

- `file_path` — tracked focus view (default: `MEMORY.md`)
- `catalog_path` — authoritative LUT (default: `memory/catalog.json`)
- `buffer_max_bytes`, `compact_when_over_bytes` — in-memory event buffer + compaction threshold (oldest-first)
- `engram_*` + `st_*` + `lt_*` — engram paths + tier budgets
- `memory_md_*` — tracked focus view caps + cadence

Related:

- Issue #13 (memory sync protocol)
- `docs/specs/systems/MemorySyncSystem.md`

### `[rate_limits.github]` and `[rate_limits.model]`

Swarm-level pacing knobs:

- concurrency caps
- minimum spacing between requests/writes
- cooldown windows on rate limit / errors

Related:

- `docs/specs/systems/RequestBudgetSystem.md`

## Related docs/specs

- `docs/dev/DIRECTOR_DEV_WORKFLOW.md`
- `docs/specs/systems/WorktreeSystem.md`
- `docs/specs/agents/DirectorDevAgent.md`
