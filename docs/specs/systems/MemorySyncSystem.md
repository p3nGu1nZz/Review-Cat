# MemorySyncSystem

## Overview

**MemorySyncSystem** defines how the ReviewCat swarm maintains shared memory without relying on a single large, high-churn tracked file as the *durable* source of truth.

It combines:

- an **in-memory bounded buffer** of normalized agent-bus events
- **oldest-first compaction** into structured **engram DTOs**
- a Director-authoritative **EngramCatalog (LUT)**
- distribution over the **agent bus** plus durable persistence under `/memory/`

`MEMORY.md` is tracked and serves as a small, bounded **shared focus view** derived from the above.

Critically:

- `/memory/**` engrams are the **shared, reviewable, durable** memory.
- `MEMORY.md` is a **shared, regeneratable, Director-managed** “current focus” view.
- Both must be **bounded** (by count and/or byte budgets) to avoid runaway growth.

## Requirements

### 1) Event ingestion (buffer)

- Normalize incoming agent-bus messages into `MemoryEvent` records (planning DTO).
- Append to a bounded buffer up to `memory.buffer_max_bytes` (or equivalent).

### 2) Oldest-first compaction

When over budget:

- compact **only the oldest events** first
- produce one or more EngramDTOs summarizing that slice
- drop/evict the compacted events from the buffer
- keep the most recent window un-compacted

### 3) Engram promotion tiers

- ST engrams capture near-term operational context.
- LT engrams capture stable conventions/lessons.

Promotion rules are policy-driven (Director) and may be informed by:

- repetition frequency
- how often an item is queried
- whether it represents an explicit decision

### 4) Director authoritative catalog

- Director maintains `EngramCatalogDTO` as the authoritative LUT.
- Director can broadcast catalog hash/version on the agent bus.
- Workers verify and request refresh if stale.

Catalog persistence (tracked in git):

- `memory/catalog.json`

### 5) Durable storage in repo

- Engrams are written to `/memory/st/<batch_id>/` and `/memory/lt/<batch_id>/`.
- The memory agent proposes engram additions via PR.
- The Director merges engram PRs into `main`.

Normative engram file path format:

- `memory/st/<batch_id>/engram_<engram_id>.json`
- `memory/lt/<batch_id>/engram_<engram_id>.json`

#### 5.1) Bounded engram policies (must-have)

To keep the repo and the swarm stable, the Director enforces limits such as:

- `memory.engram_max_bytes` — maximum size for a single EngramDTO file.
- `memory.st_max_total_bytes` / `memory.lt_max_total_bytes` — total byte budget per tier.
- `memory.st_max_files` / `memory.lt_max_files` — maximum number of engram files per tier.

When over budget, policy is oldest-first eviction at the tier level (ST first),
or promotion (ST → LT) when an item becomes stable/durable.

These limits are *not* about secrecy; they are about predictable performance,
reasonable PR sizes, and preventing “memory bloat” across parallel workers.

### 5.2) MEMORY.md focus view (Director-managed, tracked)

The Director SHOULD maintain a tracked `MEMORY.md` file as a “current focus” buffer:

- derived primarily from **recent, high-signal ST engrams** (plus optional recent events)
- regenerated/updated at controlled boundaries (configured via `config/dev.toml`):
  - `memory.memory_md_update_cadence` = `on_merge | periodic | hybrid`
  - `memory.memory_md_update_every_n_heartbeats` (when periodic/hybrid)
- capped by `memory.memory_md_max_bytes` and/or `memory.memory_md_max_items`
- treated as authoritative **for current focus only** (it is still derived/regeneratable; durable decisions belong in engrams)

This file exists to accelerate prompt bootstrapping (“what matters right now”),
not to store durable decisions.

#### Coordination rule (to avoid merge conflicts)

To minimize conflicts across parallel worktrees:

- Workers MUST treat `MEMORY.md` as **read-only**.
- Only the Director (or a dedicated MemoryAgent workflow) updates `MEMORY.md`,
  ideally in a small, reviewable PR with bounded diffs.

### 6) Worker synchronization with main

Workers should ingest changes that affect protocols/specs/memory:

- before declaring a task “ready”, worker updates its PR branch with the latest `main`.
- this pulls in newly merged engrams under `/memory/`.

Drift detection is enforced by the Director using worker heartbeat fields such as:

- `main_ancestor_sha`
- `catalog_hash_seen`

and agent-bus control messages such as `sync_required` / `protocol_incompatibility`
(see `docs/specs/systems/AgentBusSystem.md`).

## Interfaces

### Inputs

- Agent bus stream (hello/heartbeat/error/project state/memory events)
- `config/dev.toml` `[memory]` section

### Outputs

- Engram files under `/memory/`
- Tracked `MEMORY.md` focus view (bounded)
- Agent bus publications:
  - `EngramCatalogSnapshot`
  - optional `EngramSnapshot` payloads

## Acceptance criteria

- Memory stays within configured budget.
- Compaction always removes oldest events first.
- Engram files are versioned and verifiable (hash).
- Workers can detect stale/missing engrams via the catalog.

## Edge cases

- Burst telemetry causing rapid compaction loops (rate-limit compaction).
- Divergent worker clocks affecting event timestamps (use Director receipt time).
- Large engrams ballooning repo size (policy: cap ST/LT bytes/files; cap per-engram bytes).
- Rapid churn in `MEMORY.md` causing PR conflicts (mitigation: Director-only writes; controlled update cadence; keep diffs small).

## Notes

This spec intentionally focuses on planning-level behavior. Transport security and multi-host distribution constraints are covered by the agent-bus planning work.
