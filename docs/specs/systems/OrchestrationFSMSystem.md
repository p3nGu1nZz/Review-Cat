# OrchestrationFSMSystem (Director + Worker State Machines)

## Overview

`OrchestrationFSMSystem` defines finite state machines (FSMs) for the **dev harness Director** (`dev/harness/director.sh`) and **dev harness workers** (`dev/harness/run-cycle.sh`).

The goal is to make behavior:

- explicit (states + transitions)
- testable (state progression can be validated)
- resilient (clear recovery paths)
- observable (logs include state + reason)

This spec is for the **dev harness** swarm (self-coding). Runtime app pipeline FSMs are separate.

## Principles

1. **Every loop iteration has a state.** No “mystery behavior” hidden in ad-hoc branches.
2. **Transitions are event-driven.** Timeouts, completion, failure classifications, and external conditions are modeled as events.
3. **Retry is bounded.** Retry budgets come from `config/dev.toml`.
4. **External I/O is paced.** State transitions that require GitHub/model calls must respect `RequestBudgetSystem`.

## Director FSM

### States

- `BOOT`
  - load config, validate environment, open logs
- `RELEASE_PLAN`
  - ensure active release context exists (release branch + release PR)
  - select a batch of issues for the current release
- `DISCOVER`
  - fetch candidate work: open issues, PRD tasks
- `DISPATCH`
  - claim an issue, create branch/worktree, start worker container
- `MONITOR`
  - check worker heartbeats/status, collect results
- `RELEASE_FINALIZE`
  - invoke merge agent to merge release PR into `main`, resolve conflicts
  - verify gates + tag
- `BROADCAST`
  - broadcast `release_published` (new version) to swarm over agent bus
- `SELF_REVIEW`
  - run self-review when idle (creates new issues)
- `COOLDOWN`
  - apply backoff when resources are constrained (rate limits, repeated failures)
- `SHUTDOWN`
  - graceful teardown, stop workers, remove worktrees

### Events / triggers

- `tick` — heartbeat interval elapsed
- `release_missing` — no active release context exists
- `release_ready` — release plan complete; ready to merge release PR
- `release_published(tag, main_sha, image_tag)` — new version published
- `work_available` — eligible issues/tasks exist
- `capacity_available` — active workers < max
- `worker_completed(success|failure)`
- `worker_stale` — heartbeat TTL exceeded
- `rate_limited(resource, retry_after?)`
- `no_work` — no tasks and no active workers
- `signal(SIGINT|SIGTERM)`

### Transition sketch

- `BOOT -> DISCOVER` on successful init
- `BOOT -> RELEASE_PLAN` on successful init
- `RELEASE_PLAN -> DISCOVER` when release context is ready
- `DISCOVER -> DISPATCH` when `work_available && capacity_available`
- `DISCOVER -> SELF_REVIEW` when `no_work && no_active_workers`
- `DISPATCH -> MONITOR` after worker started and registered
- `MONITOR -> DISCOVER` after processing completions / updating state
- `MONITOR -> RELEASE_FINALIZE` on `release_ready`
- `RELEASE_FINALIZE -> BROADCAST` on successful merge/tag
- `BROADCAST -> DISCOVER` after broadcast completes
- `ANY -> COOLDOWN` on `rate_limited` or repeated transient failures
- `COOLDOWN -> DISCOVER` after cooldown expires
- `ANY -> SHUTDOWN` on termination signal

### Director state data (properties)

Director should track:

- active worker registry: `{worker_id, container_id, worktree_path, issue_number, state, last_heartbeat, retries}`
- issue claim timestamps
- per-resource cooldown timers (GitHub/model)
- last discovery snapshot (for dedupe)

### Logging requirements

Every state transition should log a single structured line including:

- from_state, to_state
- reason/event
- relevant ids (issue/pr/worker), redacted when needed

## Worker FSM

Workers execute a **single task cycle** for an issue.

### States

- `START`
  - validate args, locate worktree, prepare audit directory
- `UPGRADE_CHECKPOINT`
  - check whether a newer release/image tag is available
  - decide whether it is safe to restart now (safe point) or defer
- `SYNC_MAIN`
  - update branch with `main` (or verify up-to-date)
- `CONTEXT_LOAD`
  - load issue context (MCP/gh), load engram catalog + relevant ST/LT engrams
- `CODE`
  - invoke coding agent and write changes
- `VALIDATE`
  - run `scripts/build.sh && scripts/test.sh`
- `COMMIT`
  - git add/commit with message and traceability
- `PR_CREATE`
  - create PR, link issue (`Refs #N`)
  - worker PRs must not auto-close issues; the active release PR aggregates `Closes #...`
- `REVIEW_REQUEST`
  - invoke code-review agent / apply labels
- `DONE`
  - report success and exit
- `RETRY_WAIT`
  - backoff before retrying after transient failure
- `BLOCKED`
  - label/comment, record audit, exit non-zero

### Failure classification

Workers must classify failures as:

- **retryable:** transient network, timeouts, rate limit indications, flakey external tools
- **non-retryable:** compile errors after max retries, missing prereqs, invalid config

### Transition sketch

- `START -> UPGRADE_CHECKPOINT -> SYNC_MAIN -> CONTEXT_LOAD -> CODE -> VALIDATE`
- `VALIDATE -> COMMIT` on pass
- `COMMIT -> PR_CREATE -> REVIEW_REQUEST -> DONE`

On failure:

- if retryable and retries remaining: `ANY -> RETRY_WAIT -> CODE` (or earlier state depending on failure)
- else: `ANY -> BLOCKED`

On upgrade:

- if `release_published` is observed and the worker is at a **safe point**:
  - exit cleanly so the Director/supervisor can restart the container in the new image
- if not at a safe point:
  - defer upgrade until the next `UPGRADE_CHECKPOINT`

Safe point definition (MVP):

- not mid-commit/rebase/merge
- prefer clean working tree (`git status --porcelain` empty)
- if dirty but safe to discard, worker may reset to `HEAD` before restart

### Worker state data (properties)

- issue number, branch, attempt counter
- audit dir path
- last error classification and message (redacted)

## Acceptance criteria

- Director and worker behavior can be described entirely in terms of FSM states + transitions.
- There is a single bounded retry policy path.
- Rate limiting/backpressure routes through `COOLDOWN`/`RETRY_WAIT` rather than tight loops.
- Audit bundles include state transition logs sufficient to reconstruct failures.

## References

- `docs/specs/systems/RequestBudgetSystem.md`
- `config/dev.toml` (timeouts, retry, rate limits)
