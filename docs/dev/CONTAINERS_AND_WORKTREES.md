# Containers and Worktrees (Dev Harness Execution Model)

This document explains how the dev harness achieves safe parallelism.

## Goals

- Run many tasks in parallel without cross-talk.
- Keep each task’s filesystem mutations isolated.
- Make the system cheap when idle (scale-to-zero).

## Worktrees (isolation)

Each task runs in its **own git worktree** (separate working directory) with its own branch.

This makes it safe for multiple workers to:

- edit files
- run builds/tests
- generate audit artifacts

…without stomping on each other.

See the spec: `docs/dev/specs/systems/WorktreeSystem.md`.

## Containers (execution)

In the planned Docker-first model:

- There is **one shared image tag** used for all workers.
- Each task runs in **one worker container**.
- Exactly **one worktree** is bind-mounted into the container at `/workspace`.
- Workers are **stopped/removed when idle** (scale-to-zero).

This keeps the host clean and makes worker environments reproducible.

See also:

- Dev harness configuration: `docs/dev/CONFIGURATION.md`
- Director workflow: `docs/dev/DIRECTOR_DEV_WORKFLOW.md`
- Release cycle model: `docs/dev/specs/systems/ReleaseCycleSystem.md`
