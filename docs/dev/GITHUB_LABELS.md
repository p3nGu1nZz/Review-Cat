# GitHub Label Taxonomy (ReviewCat)

This document is the **single reference** for labels used to coordinate ReviewCat’s autonomous (and human) work on GitHub.

Labels are treated as a **state machine** that enables:

- discovery (what work exists?)
- exclusivity (who is working on it?)
- routing (which agent/persona should handle it?)
- prioritization (what is most urgent?)
- dashboards (humans can skim what’s going on)

**Primary consumers:** Director/worker agents (via GitHub MCP), and humans.

## Scope and principles

- Labels are part of the **durable coordination layer** (alongside Issues/PRs/comments).
- Workflow labels are **mutually exclusive** where noted (e.g., `agent-task` vs `agent-claimed`).
- Labels should be **predictable and boring**: consistent names, consistent semantics.
- When in doubt, prefer explicit issue comments over “hidden” label semantics.

## Workflow labels (agent coordination)

| Label | Meaning | Applied by | Removed by | Notes |
|---|---|---|---|---|
| `agent-task` | Available for pickup | Humans, Director, self-review | Director (on claim) | Backlog/discovery label |
| `agent-claimed` | Locked / in-progress | **Director only** | Director (on completion/release) | Enforces exclusivity; see claim protocol below |
| `agent-review` | Needs review | Coder/Director | Reviewer/Director | Optional routing label for review step |
| `agent-blocked` | Needs human input | Any agent | Humans/Director | Must include a blocking comment with clear next action |
| `auto-review` | Created by self-review | Self-review process | Never | Immutable “origin marker” |

### State invariants

- An issue MUST NOT have both `agent-task` and `agent-claimed`.
- An issue SHOULD have exactly one priority label.
- An issue SHOULD have at least one persona/domain label.

## Persona / domain labels

| Label | Meaning |
|---|---|
| `security` | Security findings, unsafe defaults, token handling |
| `performance` | Efficiency, allocations, algorithmic complexity |
| `architecture` | Structure, interfaces, layering, boundaries |
| `testing` | Tests, harnesses, determinism, coverage |
| `docs` | Documentation and specifications |

## Priority labels

| Label | Meaning |
|---|---|
| `priority-critical` | Blocks progress; should be addressed first |
| `priority-high` | Important; do soon |
| `priority-medium` | Normal priority |
| `priority-low` | Nice-to-have |

## Type labels (optional, Phase 2+)

These are optional but encouraged once the backlog grows.

| Label | Meaning |
|---|---|
| `type:bug` | Something isn’t working |
| `type:feature` | New capability |
| `type:refactor` | Code improvement without new behavior |
| `type:chore` | Maintenance (deps, CI, formatting) |

## Issue lifecycle (label state machine)

Typical issue flow:

```
[Open] + agent-task
  -> (Director claims)
[Open] + agent-claimed
  -> (work done; PR created)
[Open] + agent-review (optional)
  -> (PR merged)
[Closed]

OR

[Open] + agent-claimed
  -> (agent blocked)
[Open] + agent-blocked
  -> (human resolves + unblocks)
[Open] + agent-claimed
```

## PR lifecycle (routing)

PRs can be routed with labels (or just review requests). If using labels:

```
[PR opened]
  -> agent-review
  -> (review complete)
  -> (merge)
```

## Issue claim locks (exclusive work) 🔒

**Goal:** Ensure no two workers implement the same issue concurrently.

### Rules

- **Only the Director** is allowed to claim issues (workers must never self-claim).
- Claiming is a *lock acquisition*:
  - remove `agent-task`
  - add `agent-claimed`
  - add a machine-parseable claim comment (format below)
- The Director MUST re-read the issue after claiming and confirm the lock is held.

### Claim comment format (machine-parseable)

The claim comment MUST start with the prefix `ReviewCat-Claim:` and include a minimal YAML-ish payload.

Example:

```text
ReviewCat-Claim:
  claimed_by: director
  run_id: <run_id>
  worker_id: <worker_id>
  claimed_at: 2026-02-11T23:59:59Z
  release_branch: feature/release-20260211-235959Z
```

Notes:

- `run_id` is the Director run identifier (unique per swarm run).
- `worker_id` is the logical worker id (or container name).
- `release_branch` is optional for non-release workflows, but preferred.

### Lock timeout / reclaim policy

A claim is considered **stale** if there has been no progress signal within a configured window (time-based TTL and/or max-behind heartbeat TTL).

When stale, the Director MAY reclaim by:

- adding a reclaim comment:
  - `ReviewCat-Reclaim: reason: stale_claim ...`
- removing `agent-claimed`
- adding `agent-task`

The reclaim policy is intentionally conservative in MVP: prefer warning + comment first.

### Director single-instance guard (optional MVP)

To avoid two Directors competing, optionally use a dedicated “director lock” issue (or branch-based lock). This is documented in Issue #11 planning.

## Recommended query patterns

- Open backlog:
  - label: `agent-task`
- Work in progress:
  - label: `agent-claimed`
- Human attention:
  - label: `agent-blocked`

## Cross-references

- `PLAN.md` (coordination model and release cycles)
- `docs/DIRECTOR_DEV_WORKFLOW.md` (operational sequence)
- `docs/specs/systems/GitHubOpsSystem.md` (API operations)
- Issue #10 (this document)
- Issue #11 (claim locks)
