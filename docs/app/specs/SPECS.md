# Runtime App Specs Index

This index groups specs by **runtime app** concern.

> Note: Many specs currently live under `docs/specs/` while the docs structure is being migrated.
> The authoritative split is **dev harness vs runtime app**; physical file moves can happen incrementally.

## Components

- [`RunConfig`](../../specs/components/RunConfig.md)
- [`ReviewInput`](../../specs/components/ReviewInput.md)
- [`ReviewFinding`](../../specs/components/ReviewFinding.md)
- [`PromptRecord`](../../specs/components/PromptRecord.md)
- [`AuditRecord`](../../specs/components/AuditRecord.md)

## Entities

- [`AuditIdFactory`](../../specs/entities/AuditIdFactory.md)
- [`PromptFactory`](../../specs/entities/PromptFactory.md)

## Systems (runtime app)

- [`CLIFrontend`](../../specs/systems/CLIFrontend.md)
- [`RepoDiffSystem`](../../specs/systems/RepoDiffSystem.md)
- [`CopilotRunnerSystem`](../../specs/systems/CopilotRunnerSystem.md)
- [`PersonaReviewSystem`](../../specs/systems/PersonaReviewSystem.md)
- [`SynthesisSystem`](../../specs/systems/SynthesisSystem.md)
- [`PatchApplySystem`](../../specs/systems/PatchApplySystem.md)
- [`LoggingSystem`](../../specs/systems/LoggingSystem.md)
- [`DirectorRuntimeSystem`](../../specs/systems/DirectorRuntimeSystem.md)
- [`PersonaReviewSystem`](../../specs/systems/PersonaReviewSystem.md)

## Systems (shared)

- [`GitHubOpsSystem`](../../specs/systems/GitHubOpsSystem.md)
- [`RequestBudgetSystem`](../../specs/systems/RequestBudgetSystem.md)
- [`AuditStoreSystem`](../../specs/systems/AuditStoreSystem.md)

## Related docs

- App configuration: `docs/app/CONFIGURATION.md`
