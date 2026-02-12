# /memory — Shared Engram Store (Tracked)

This directory contains **durable shared agent memory** as versioned, structured **engram DTOs**.

## Why this exists

- The swarm produces a high-volume stream of agent-bus events (heartbeats, errors, state changes, decisions).
- Keeping shared memory in a single frequently-updated tracked file does not scale across parallel worktrees.
- Instead, older event slices are compacted into **engram files** that can be committed, reviewed, merged, and fetched like normal repo artifacts.

## Structure

The shared durable memory layout is intentionally **append-friendly** and **merge-friendly**.

- `memory/catalog.json` — the authoritative **EngramCatalogDTO** (Director-maintained LUT)
- `memory/st/<batch_id>/engram_<engram_id>.json` — short-term engrams (recent, high relevance)
- `memory/lt/<batch_id>/engram_<engram_id>.json` — long-term engrams (stable conventions and lessons)

In addition to these durable engrams, the repo root contains:

- `MEMORY.md` — a **tracked, bounded shared focus view** maintained by the Director and derived from recent high-signal ST/LT context.

### Naming rules (normative)

- `<batch_id>` MUST be a UTC timestamp:
	- format: `YYYYMMDD-HHMMSSZ` (example: `20260211-231715Z`)
	- used as the **directory name** and should match the EngramDTO `version`
- `<engram_id>` MUST be filename-safe and stable.
	- recommended: a ULID with an `e_` prefix (example: `e_01J1Z2K3M4N5P6Q7R8S9T0V1W2`)
	- allowed characters: `A-Z a-z 0-9 _ -`

### Example

```text
memory/
├── README.md
├── catalog.json
├── st/
│   ├── README.md
│   ├── 20260211-231715Z/
│   │   ├── engram_e_01J1Z2K3M4N5P6Q7R8S9T0V1W2.json
│   │   └── engram_e_01J1Z2K3M4N5P6Q7R8S9T0V1W3.json
│   └── 20260210-090000Z/
│       └── engram_e_01J1Z2K3M4N5P6Q7R8S9T0V1W0.json
└── lt/
		├── README.md
		└── 20260101-000000Z/
				└── engram_e_01HZ0Y8X7W6V5U4T3S2R1Q0P9O.json
```

> Note: we intentionally standardize on timestamp batch ids (not release tags) to keep directory ordering predictable and avoid ambiguous sorting.

## Governance

- Engrams should be treated as **append-only** once merged (prefer new versions over edits).
- The Director maintains an authoritative **engram catalog (LUT)** mapping ids → hashes/paths/versions.
- Workers should stay in sync with `main` to pull the latest engrams into their worktrees.
	- In the dev harness, the Director detects drift via worker heartbeats (e.g., `catalog_hash_seen`) and can require sync via agent-bus messages (see `docs/specs/systems/AgentBusSystem.md`).

See:
- `AGENT.md` (memory model)
- `config/dev.toml` `[memory]` keys
- GitHub Issue #13 (planning)
