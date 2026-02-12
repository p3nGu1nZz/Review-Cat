# Legacy Specs Index (Migration)

This repository is migrating from a single flat spec tree (`docs/specs/`) to a **two-track** hierarchy:

- **Dev harness specs** (agents + harness systems): [`docs/dev/specs/SPECS.md`](../dev/specs/SPECS.md)
- **Runtime app specs** (components/entities/systems): [`docs/app/specs/SPECS.md`](../app/specs/SPECS.md)

During this transition, many spec files still live physically under `docs/specs/`. Use the dev/app indices above for navigation.

## Legacy directories

- Agents: [`docs/specs/agents/`](agents/)
- Components: [`docs/specs/components/`](components/)
- Entities: [`docs/specs/entities/`](entities/)
- Systems: [`docs/specs/systems/`](systems/)

## Creating new specs

Prefer placing new specs under the appropriate tree:

- `docs/dev/specs/` for dev harness concerns
- `docs/app/specs/` for runtime app concerns

The spec template is at [`docs/specs/_template.md`](_template.md).
