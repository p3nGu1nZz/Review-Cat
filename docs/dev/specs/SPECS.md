# Dev Harness Specs Index

This index groups specs by **dev harness** concern.

> Note: Many specs currently live under `docs/specs/` while the docs structure is being migrated.
> The authoritative split is **dev harness vs runtime app**; physical file moves can happen incrementally.

## Agents

- [`DirectorDevAgent`](../../specs/agents/DirectorDevAgent.md)
- [`CodingAgent`](../../specs/agents/CodingAgent.md)
- [`MemoryAgent`](../../specs/agents/MemoryAgent.md)
- [`RoleAgents`](../../specs/agents/RoleAgents.md)

## Systems (dev harness)

- [`WorktreeSystem`](../../specs/systems/WorktreeSystem.md)
- [`AgentBusSystem`](../../specs/systems/AgentBusSystem.md)
- [`MemorySyncSystem`](../../specs/systems/MemorySyncSystem.md)
- [`OrchestrationFSMSystem`](../../specs/systems/OrchestrationFSMSystem.md)
- [`ReleaseCycleSystem`](../../specs/systems/ReleaseCycleSystem.md)
- [`SelfReviewSystem`](../../specs/systems/SelfReviewSystem.md)
- [`CodingAgentSystem`](../../specs/systems/CodingAgentSystem.md)
- [`SwarmVisualizerSystem`](../../specs/systems/SwarmVisualizerSystem.md)

## Systems (shared)

Some systems are used in both dev and runtime contexts:

- [`GitHubOpsSystem`](../../specs/systems/GitHubOpsSystem.md)
- [`RequestBudgetSystem`](../../specs/systems/RequestBudgetSystem.md)
- [`AuditStoreSystem`](../../specs/systems/AuditStoreSystem.md)

## Components (dev harness)

- [`StateFile`](../../specs/components/StateFile.md)
- [`EngramDTO`](../../specs/components/EngramDTO.md)
- [`EngramCatalogDTO`](../../specs/components/EngramCatalogDTO.md)

## Related docs

- Director workflow: `docs/dev/DIRECTOR_DEV_WORKFLOW.md`
- Containers + worktrees: `docs/dev/CONTAINERS_AND_WORKTREES.md`
