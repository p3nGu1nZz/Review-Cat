# WorktreeSystem

## Overview

`WorktreeSystem` manages git worktree lifecycle for parallel agent execution.
Each coding agent or implementer works in an isolated worktree so that
multiple agents can compile, test, and commit independently without conflicts.

Worktrees are created in the parent directory of the main repository.

## Requirements

1. Must create worktrees with unique branch names.
2. Must place worktrees in the parent directory (e.g., `../Review-Cat-agent-<N>-<ts>/`).
3. Must support teardown (removal) of completed worktrees.
4. Must list active worktrees.
5. Must enforce a maximum concurrent worktree limit (`MAX_WORKERS`).
6. Must handle cleanup of orphaned worktrees on Director restart.
7. Must tolerate the presence of a gitignored root `STATE.json` file inside worktrees.
8. Must never treat `STATE.json` as durable coordination (it is local cached state only).

## Interfaces

- `create(branch_name) -> worktree_path`
- `teardown(worktree_path) -> void`
- `list() -> worktree_info[]`
- `count_active() -> int`
- `cleanup_orphans() -> void`

## Implementation

```bash
# dev/harness/worktree.sh

create() {
    BRANCH=$1
    WORKTREE_DIR="../Review-Cat-${BRANCH//\//-}-$(date +%s)"
    git worktree add "$WORKTREE_DIR" -b "$BRANCH"
    echo "$WORKTREE_DIR"
}

teardown() {
    WORKTREE_DIR=$1
    git worktree remove "$WORKTREE_DIR" --force
}

list() {
    git worktree list --porcelain
}
```

## Acceptance criteria

- Worktrees are created in the parent directory with unique names.
- `MAX_WORKERS` limit is respected.
- Teardown removes the worktree directory and git reference.
- Orphaned worktrees from crashed workers are cleaned up.

## `STATE.json` (local cached state)

Workers and the Director may create a root-level `STATE.json` file in any
checkout (main worktree or worker worktrees).

- The file is gitignored and never committed.
- It is created lazily if missing.
- It can be used to detect first-run vs resume after container restart.

Worktree teardown implicitly removes `STATE.json` because the directory is
removed.

Recommended minimum fields for `STATE.json` are defined in:

- `docs/specs/components/StateFile.md`

## Test cases

- Create and teardown a worktree.
- Verify branch naming convention.
- Verify MAX_WORKERS enforcement.
- Verify orphan cleanup on restart.

## Edge cases

- Parent directory not writable.
- Branch name collision (two agents pick same branch).
- Worktree removal fails (files locked or in use).
- Git worktree command not available (old git version).
- A worker restarts mid-task and must resume using local cached state.

## Non-functional constraints

- Worktree names must be deterministic for debugging.
- No worktree should modify the main checkout.
- Cleanup must be safe (never delete the main worktree).
