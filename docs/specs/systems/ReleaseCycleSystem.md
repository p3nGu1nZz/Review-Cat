# ReleaseCycleSystem (Dev Harness)

## Overview

`ReleaseCycleSystem` defines how the **dev harness** batches work into a
**release cycle**, merges work into a single release branch/PR, finalizes the
release via a dedicated merge agent, and broadcasts the new version to the
swarm.

This spec applies to the **self-coding dev harness** (Director + workers), not
the runtime app pipeline.

## Goals

1. Make “what ships together” explicit.
2. Reduce merge churn on `main` by batching many worker PRs into one release PR.
3. Provide a deterministic, resumable flow across container restarts.
4. Enable safe mid-swarm upgrades via version broadcasts + safe restart points.

## Key concepts

### Release branch + release PR

- Release branch naming (normative):
  - `feature/release-<release_id>`
- Release PR (normative):
  - `feature/release-<release_id>` → `main`

### Release plan

A release plan is the set of GitHub issue numbers selected for the current
release.

- Durable representation:
  - the release PR description MUST list included issues
  - the release PR MUST include aggregated issue closures (e.g., `Closes #12`)
- Local cached representation:
  - `STATE.json` MAY cache the active release context and selected issue list

### Worker PR targeting

Workers SHOULD:

- create PRs targeting the active release branch
- use `Refs #<issue>` (not `Closes #<issue>`) so issues close when the release
  PR is merged into `main`

## Local cached state (`STATE.json`)

A gitignored root-level `STATE.json` is used for **local cached state** only.

Normative rules:

- MUST be gitignored and never committed.
- MUST be created lazily if missing.
- MUST be safe to delete at any time (system should recreate it).

Recommended minimum fields (schema is intentionally flexible for MVP):

- `schema_version`
- `created_at`, `updated_at`
- `active_release`: `{ release_id, branch, pr_number?, base_main_sha? }`
- `last_seen_main_sha`
- `last_seen_image_tag`

## Interfaces (conceptual)

- `ensure_release_context() -> {release_id, release_branch, release_pr}`
  - create or refresh release branch and release PR
- `select_release_batch(candidates) -> issue_numbers[]`
  - choose which issues are in the current release
- `merge_worker_pr(pr) -> void`
  - merge worker PR into release branch (after validation)
- `finalize_release() -> {tag, main_sha}`
  - invoke merge agent to merge release PR into `main`
  - resolve conflicts
  - run validation gates
  - create/verify tag
- `broadcast_release(tag, main_sha, image_tag) -> void`
  - emit `release_published` to agent bus

## Release finalization (merge agent)

## Release readiness (Director decision)

The Director SHOULD consider a release “ready to finalize” when:

1. All issues in the active release plan are either:
  - merged into the release branch, or
  - explicitly deferred with a written reason (comment/label), and removed from the release plan.
2. The release branch is green on the validation gate(s) (build/test once those exist).
3. There are no active workers still targeting the release branch (or they are at a safe checkpoint).
4. The Director has recorded the intended image tag for this release (single shared image tag) and can broadcast it after merge.

This is intentionally conservative: prefer fewer, well-validated releases over frequent churn.

When the release plan is complete, the Director invokes a dedicated
**merge agent expert**.

Responsibilities:

1. Merge the release PR into `main`.
2. Resolve merge conflicts (if any) using:
   - release plan context
   - recent engrams / `MEMORY.md` focus view
3. Ensure build/test gates pass.
4. Create/verify the release tag.
5. Notify the Director with a structured completion report.

## Upgrade protocol (workers)

When `release_published` is broadcast:

- Workers MAY restart into the new shared image tag only at **safe points**:
  - between commits
  - not mid-rebase/merge
  - ideally with a clean working tree
- If mid-edit and safe to discard, worker MAY reset to `HEAD` before restart.
- If mid-critical git operation, worker MUST finish that operation first.

## Acceptance criteria

- A release branch and release PR are always present while the Director is
  active.
- Worker PRs target the release branch and do not close issues directly.
- The merge agent merges the release PR into `main` and creates/verifies a tag.
- After finalization, the swarm receives a `release_published` broadcast.
- The system is resilient to container restart using `STATE.json` for cache.

## References

- `AGENT.md` (swarm contract)
- `docs/DIRECTOR_DEV_WORKFLOW.md` (workflow)
- `docs/specs/systems/OrchestrationFSMSystem.md` (state machines)
- GitHub Issue: #14 (Docker image lifecycle + release updates)
