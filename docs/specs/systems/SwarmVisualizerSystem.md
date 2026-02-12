# SwarmVisualizerSystem

## Overview

The **SwarmVisualizerSystem** is an optional **operator UI client** that connects to the ReviewCat **agent bus** (socket pub/sub) and renders a live 3D visualization of the swarm:

- workers (containers / processes)
- tasks (issues / PRD items)
- message edges (telemetry, errors, memory updates)

The visualizer is intended to run on the operator’s machine (Windows or Linux) and connect to a swarm running in Linux Docker (local or remote). It is a planning artifact: this spec defines behavior and interfaces; implementation must trace back to it.

## Requirements

### Functional requirements

1. **Connect/disconnect**
  - Connect to agent bus endpoint (host/port) defined by `config/dev.toml` (`[agent_bus].listen_addr` / `[agent_bus].listen_port`).
   - Show connection status and last-received message time.
  - Participate in agent-bus version negotiation (see `AgentBusSystem`):
    - send `protocol_hello.v1` as `sender.role = operator`
    - expect `protocol_welcome.v1` (or handle `protocol_incompatibility.v1`)

2. **Live swarm graph rendering (3D)**
   - Render nodes for:
     - Director
     - worker instances (containers)
     - tasks (issue numbers / PRD items)
   - Render edges for:
     - worker → task assignment
     - worker → Director heartbeat
     - error/report events
   - Provide filtering and search (by issue number, worker id, label/status).

3. **Inspector panels**
   - Selecting a node shows:
     - identifiers (worker id, container id, branch/worktree, issue id)
     - state (running/exited/blocked, heartbeat TTL, retry counters)
     - last error payload (if any)
   - Selecting an edge shows:
     - channel/topic
     - message rates and last payload summary

4. **Operator controls**
   - Start/stop swarm (Director) where supported by the control plane.
   - Pause/unpause individual workers.
   - Request a ProjectState snapshot refresh.

5. **Camera and interaction**
   - Free-fly camera (WASD + mouse look) and orbit mode.
   - Zoom, pan, focus-on-selected.

6. **UI chrome + overlays (ToolUI)**
   - Render a minimal “tool UI” overlay without external UI frameworks:
     - draw primitives (lines, rect fill, rect outline)
     - text via a shared bitmap glyph font (see `ToolUISystem`)
     - colors set programmatically via hex codes (e.g., `#RRGGBB`, `#RRGGBBAA`)
   - Provide two single-line status bars:
     - **Top status bar:** hotkey/helper text for function keys (F1–F12)
     - **Bottom status bar:** focused panel/view state + key status info
   - Support explicit UI layering with a **z-index** concept:
     - world/scene viewport (graph)
     - overlay panels (inspector, filters, logs)
     - modal confirmations (highest z)

### Non-functional requirements

- **Cross-platform UI stack:** SDL3 for windowing/input; custom ToolUI for 2D overlay primitives + bitmap font. 3D rendering backend is an implementation detail (e.g., SDL3 GPU API or OpenGL) and must be abstracted.
- **Responsiveness:** visualizer remains interactive under bursty telemetry.
- **Safety:** operator controls must be explicit and gated (no accidental destructive actions).
- **Observability:** record a local log of received telemetry + user actions (no secrets).

## Interfaces

### Inputs

- Agent bus messages (framed + enveloped). At minimum (message types defined in `docs/specs/systems/AgentBusSystem.md`):
  - `heartbeat.v1`
  - `worker_error.v1`
  - `project_state_snapshot.v1`

  Optional-but-useful for richer UI:

  - `protocol_hello.v1` / `protocol_welcome.v1` (connection/debug panels)
  - `sync_required.v1` / `protocol_incompatibility.v1` (drift/compat alerts)
  - `engram_announce.v1` / `engram_catalog_snapshot.v1` (memory activity panels)

### Outputs

- Optional control-plane commands (request snapshot, pause worker, stop swarm).
- Local operator logs.

### Configuration

- Reads:
  - agent bus address/ports from `config/dev.toml`
  - heartbeat TTL from `config/dev.toml` (`[timeouts].worker_heartbeat_ttl_seconds`) for “stale worker” UI state
  - optional UI defaults (camera speed, theme) from a UI config section (future)

## Acceptance criteria

- With a running Director and at least one worker:
  - the visualizer connects and shows the Director node and the worker node
  - worker heartbeat TTL updates in the inspector
  - selecting nodes/edges shows structured details
- When a worker emits a structured error message:
  - the visualizer surfaces an alert and allows drilling into the payload
- Operator controls are present but safe:
  - any stop/pause action requires explicit confirmation

## Test cases

- Connect to a local test bus endpoint and receive synthetic heartbeat messages.
- Simulate a worker disconnect and verify the UI marks it stale after TTL.
- Inject a burst of messages and verify frame rate and input responsiveness remain acceptable.

## Edge cases

- Network interruption / reconnect loops.
- Clock skew between host and swarm affecting TTL display.
- Partial telemetry (e.g., missing container id) should degrade gracefully.

## Notes

- This spec assumes the existence of an agent bus and ProjectState DTOs; canonical message types and envelope framing are defined in `docs/specs/systems/AgentBusSystem.md`.
- The visualizer is an operator tool for the dev harness first; it may later be integrated into the runtime app UI if it remains valuable.
