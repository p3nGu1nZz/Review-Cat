# Prompt Cookbook

This is a curated set of prompt patterns used by ReviewCat.

> **Context:** Prompts are invoked via `copilot -p "prompt"` subprocess calls —
> from **bash scripts** (dev harness) and from the **C++ binary** (runtime app).
> See [PLAN.md](../../PLAN.md) §4.3 for integration details.
>
> **MCP config:** All agent invocations include `--mcp-config dev/mcp/github-mcp.json`
> which configures the GitHub MCP Server (remote MCP preferred; or native `github-mcp-server stdio` on host / container-bundled).
> See [PLAN.md](../../PLAN.md) §3.1 for deployment options.

The intent is consistency and auditable prompt engineering.

## Persona prompt contract

All persona prompts must request output in strict JSON, as an array of findings.

Each finding:

- file
- line_start
- line_end
- severity: critical|high|medium|low|nit
- title
- description
- rationale
- suggested_fix
- suggested_patch (optional, unified diff)
- confidence: 0.0..1.0

## Persona: security

Prompt pattern:

- Input: diff hunks and minimal context.
- Output: vulnerabilities, exploitability, unsafe defaults, data leak risks.
- Require: at least one concrete fix suggestion per finding.

## Persona: performance

Prompt pattern:

- Focus on algorithmic complexity, hot paths, allocations, I/O.
- Require: explain impact and conditions.

## Persona: architecture

Prompt pattern:

- Focus on module boundaries, naming, dependency direction.
- Require: propose refactors only if they reduce complexity.

## Persona: testing

Prompt pattern:

- Identify missing tests and propose concrete test cases.

## Persona: docs

Prompt pattern:

- Identify missing usage docs and confusing behaviors.

## Synthesis prompt

Prompt pattern:

- Input: all persona JSON outputs.
- Output:
  - unified review markdown
  - merged finding list with dedupe rationale
  - action plan JSON suitable for GitHub issues

## Patch planning prompt

Prompt pattern:

- Input: prioritized findings.
- Output: small safe patches.
- Constraints:
  - avoid broad refactors
  - prefer tests first
  - no unrelated formatting changes

## DirectorDev Planning Prompt

Prompt pattern:

- Input: a spec file from `docs/specs/`.
- Output:
  - task graph
  - role assignments
  - exact acceptance criteria mapping

### Planning issue checklist (governance)

When creating or updating a **planning** GitHub Issue that changes swarm behavior
(workflow, protocols, roles, orchestration, memory model), include an explicit
checklist item:

- **[ ] Update `AGENT.md` contract** (and keep it consistent with the relevant specs)

In practice, swarm-behavior changes usually also require updating one or more of:

- the relevant spec(s) under `docs/specs/`
- `PLAN.md` / `TODO.md` if the change affects phases, workflow, or release tasks
- `docs/dev/DIRECTOR_DEV_WORKFLOW.md` if it changes how the Director/workers operate

## DirectorDev Heartbeat Prompts

The Director (recommended entry: `dev/scripts/daemon.sh`; underlying loop:
`dev/harness/director.sh`) uses these prompt patterns when delegating to role
agents:

### Implementer

```
copilot -p @dev/agents/implementer.md \
  "Implement the following spec: <spec content>. \
   Write C++17/20 code under app/src/. Use nlohmann/json for JSON, \
   toml++ for config, Catch2 for tests. Follow CMake build conventions." \
  --allow-tools write
```

### QA

```
copilot -p @dev/agents/qa.md \
  "Write Catch2 tests for the spec: <spec content>. \
   Tests go under app/tests/. Use record/replay fixtures where Copilot \
   CLI responses are needed." \
  --allow-tools write
```

### Security

```
copilot -p @dev/agents/security.md \
  "Security review the changes for: <spec content>. \
   Check for: buffer overflows, unsafe subprocess calls, credential \
   exposure, path traversal."
```

### Code Review

```
copilot -p @dev/agents/code-review.md \
  "Review this diff: <git diff output>. \
   Check for: C++ best practices, memory safety, error handling, \
   consistent naming, CMake target correctness."
```

