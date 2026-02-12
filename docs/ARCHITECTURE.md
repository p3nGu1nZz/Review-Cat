# Architecture Overview

ReviewCat is designed as a **two-part system**:

1. **Runtime app** (`app/`): the end-user CLI/daemon/UI.
2. **Dev harness** (`dev/`): the self-improving development loop (Director + workers).

The repo’s “meta loop” is:

> **bootstrap → dev → app → dev → app → …**

## Runtime app (end-user)

The runtime app runs reviews on diffs or PRs using persona prompts, synthesizes findings, and optionally opens issues/PRs in the user’s repository.

Specs live under `docs/app/specs/`.

## Dev harness (self-building)

The dev harness runs as a Director daemon that orchestrates worker tasks against isolated git worktrees (and, in the planned model, worker containers).

Specs live under `docs/dev/specs/`.

## Navigation

- Docs landing page: [`docs/INDEX.md`](INDEX.md)
- Project plan: [`PLAN.md`](../PLAN.md)
- Task list: [`TODO.md`](../TODO.md)
- Swarm operating contract: [`AGENT.md`](../AGENT.md)
