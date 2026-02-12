# StateFile (`STATE.json`)

## Overview

`STATE.json` is a **gitignored local cached state** file stored at the repo root in any checkout (main worktree or worker worktree).

It exists to make the dev harness more resilient:

- detect **first-run vs resume** after container/supervisor restart
- cache active release context (release id/branch/PR)
- cache last-seen `main` SHA and memory catalog hash
- avoid brittle “stateless” assumptions during orchestration

`STATE.json` is an optimization only.

- It MUST be safe to delete at any time.
- It MUST be created lazily if absent.
- It MUST NOT be treated as durable coordination.

Durable coordination remains: GitHub Issues/PRs + tracked `/memory/**` engrams + `memory/catalog.json`.

## Location and governance (normative)

- Path: repo root `STATE.json`
- Git policy: MUST be gitignored and never committed

## Minimal schema (recommended)

This is a recommended minimum schema. Fields may be added over time.

```json
{
	"schema_version": "reviewcat-state/v1",
	"created_at": "2026-02-12T00:00:00Z",
	"updated_at": "2026-02-12T00:00:00Z",
	
	"active_release": {
		"release_id": "20260212-000000Z",
		"release_branch": "feature/release-20260212-000000Z",
		"release_pr_number": 123
	},
	
	"last_seen": {
		"main_sha": "bbadafb...",
		"catalog_hash": "sha256:...",
		"image_tag": "reviewcat-dev:main"
	},
	
	"director": {
		"run_id": "run_20260212-000000Z",
		"last_heartbeat_at": "2026-02-12T00:00:00Z"
	},
	
	"worker": {
		"worker_id": "worker-3-1707600000",
		"issue_number": 21,
		"branch_name": "agent/21-1707600000",
		"stage": "CODE",
		"attempt": 1
	}
}
```

### Field notes

- `schema_version` MUST be present.
- `created_at` SHOULD be written once and then preserved.
- `updated_at` SHOULD be updated whenever the file is written.
- `active_release` MAY be omitted if no release context exists.
- `worker` SHOULD be omitted in the Director’s main worktree; it is most useful inside worker worktrees.

## Invariants

- The file MUST be writable by the agent process in its container/worktree.
- The file MUST NOT include secrets (tokens, credentials). If a value looks sensitive, it must be redacted or omitted.
- Readers MUST handle missing fields (forward-compatible parsing).

## References

- Issue #22
- `AGENT.md` (local cached state)
- `docs/DIRECTOR_DEV_WORKFLOW.md` (resume + release context)
- `docs/specs/systems/WorktreeSystem.md`
